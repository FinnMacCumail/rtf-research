# Phase 2: NetBox MCP Server Integration

## NetBoxLabs Official MCP Server

**Repository**: [https://github.com/netboxlabs/netbox-mcp-server](https://github.com/netboxlabs/netbox-mcp-server)

The NetBoxLabs MCP server provides a simple, read-only Model Context Protocol interface to NetBox infrastructure data, enabling AI agents to query network documentation through standardized tools.

## Architecture Overview

### Generic Tool Design

Unlike specialized tool-per-object-type approaches (get_device_by_id, get_site_by_id, list_devices, etc.), the NetBoxLabs server implements **3 generic tools** that handle all NetBox object types:

1. **`get_objects`** - Generic retrieval with filtering and pagination
2. **`get_object_by_id`** - Detailed single object lookup
3. **`get_changelogs`** - Change history tracking

**Benefits**:
- Dramatically reduced token overhead (3 tool schemas vs 100+)
- Simplified agent orchestration
- Consistent interface across all object types
- Easy to extend for new NetBox object types

### Field Filtering for Token Optimization

The server supports field filtering to retrieve only needed data:

```python
# Instead of retrieving full device objects (1000+ tokens each)
get_objects(
    object_type="dcim.devices",
    fields=["id", "name", "status", "site"],  # Only 4 fields
    filters={"site": "DM-Scranton"}
)
```

**Token Savings**:
- Full device object: ~1000-2000 tokens
- Filtered device (4 fields): ~50-100 tokens
- **90% token reduction** for typical queries

### Read-Only Safety

All operations are read-only and idempotent:
- No write/update/delete capabilities
- Safe for AI agent exploration
- No risk of accidental infrastructure changes
- Suitable for production NetBox instances

## Tool Usage Patterns

### 1. Generic Object Retrieval (`get_objects`)

Query any NetBox object type with filters:

```python
# Devices at a specific site
get_objects(
    object_type="dcim.devices",
    filters={"site": "DM-Scranton"},
    fields=["id", "name", "device_type", "status"],
    limit=50
)

# IP addresses in a prefix
get_objects(
    object_type="ipam.ip_addresses",
    filters={"prefix": "10.0.0.0/24"},
    fields=["id", "address", "dns_name", "status"],
    limit=100
)

# VLANs with specific name pattern
get_objects(
    object_type="ipam.vlans",
    filters={"name__contains": "prod"},
    fields=["id", "vid", "name", "site"]
)
```

**Supported Object Types**:
- **DCIM**: devices, sites, racks, interfaces, cables, power feeds, etc.
- **IPAM**: IP addresses, prefixes, VLANs, VRFs, aggregates
- **Virtualization**: virtual machines, clusters, interfaces
- **Tenancy**: tenants, contacts, contact assignments
- **Circuits**: circuits, providers, circuit types

### 2. Single Object Detail (`get_object_by_id`)

Retrieve complete details for a specific object:

```python
# Get full device details
get_object_by_id(
    object_type="dcim.devices",
    object_id=123,
    fields=["id", "name", "device_type", "site", "primary_ip4", "status", "serial"]
)

# Get specific VLAN information
get_object_by_id(
    object_type="ipam.vlans",
    object_id=456,
    fields=["id", "vid", "name", "description", "site", "tenant"]
)
```

### 3. Change History (`get_changelogs`)

Track object modifications over time:

```python
# Get recent changes for a device
get_changelogs(
    object_type="dcim.devices",
    object_id=123,
    limit=20
)

# Audit trail for IP address assignments
get_changelogs(
    object_type="ipam.ip_addresses",
    object_id=789
)
```

## Integration with Agent Frameworks

The NetBoxLabs MCP server serves as **shared infrastructure** for different agent frameworks:

### Phase 4A: Deepagents (LangChain)
```python
# Deepagents dynamically wraps MCP tools
from deepagents import async_create_deep_agent

agent = await async_create_deep_agent(
    name="NetBox Analyst",
    tools=discover_netbox_mcp_tools(),  # Wraps get_objects, get_object_by_id, get_changelogs
    enable_cache=True
)
```

### Phase 4B: Claude Agent SDK
```python
# Claude SDK uses MCP protocol directly
options = ClaudeAgentOptions(
    mcp_servers={
        "netbox": {
            "command": "python",
            "args": ["-m", "netbox_mcp_server"],
            "env": {"NETBOX_URL": url, "NETBOX_TOKEN": token}
        }
    },
    allowed_tools=["netbox_get_objects", "netbox_get_object_by_id", "netbox_get_changelogs"]
)
```

**Key Point**: Both frameworks use the same 3 MCP tools; the difference is in orchestration approach (custom vs managed).

## Configuration

### Environment Variables
```bash
NETBOX_URL=https://netbox.example.com
NETBOX_TOKEN=your_api_token_here
```

### Docker Deployment
```dockerfile
FROM python:3.11-slim
RUN pip install netbox-mcp-server
ENV NETBOX_URL=https://netbox.example.com
ENV NETBOX_TOKEN=${NETBOX_TOKEN}
CMD ["python", "-m", "netbox_mcp_server"]
```

### MCP Server Configuration
```json
{
  "mcpServers": {
    "netbox": {
      "command": "python",
      "args": ["-m", "netbox_mcp_server"],
      "env": {
        "NETBOX_URL": "https://netbox.example.com",
        "NETBOX_TOKEN": "your_token"
      }
    }
  }
}
```

## Query Examples

### Cross-Domain Infrastructure Queries

**"Show all devices at site DM-Scranton with their IP addresses"**
```python
# Step 1: Get devices
devices = get_objects(
    object_type="dcim.devices",
    filters={"site__name": "DM-Scranton"},
    fields=["id", "name", "primary_ip4", "status"]
)

# Step 2: Get IP details (if needed)
for device in devices:
    if device.primary_ip4:
        ip_detail = get_object_by_id(
            object_type="ipam.ip_addresses",
            object_id=device.primary_ip4.id,
            fields=["address", "dns_name", "description"]
        )
```

**"What racks are available in the Akron data center?"**
```python
racks = get_objects(
    object_type="dcim.racks",
    filters={"site__name": "Akron", "status": "available"},
    fields=["id", "name", "u_height", "facility_id"]
)
```

**"List all active circuits for tenant Dunder Mifflin"**
```python
circuits = get_objects(
    object_type="circuits.circuits",
    filters={"tenant__name": "Dunder Mifflin", "status": "active"},
    fields=["id", "cid", "provider", "type", "description"]
)
```

## Performance Characteristics

### Token Efficiency

| Query Type | Without Field Filtering | With Field Filtering | Reduction |
|------------|------------------------|---------------------|-----------|
| Single device | ~1500 tokens | ~150 tokens | 90% |
| 50 devices | ~75,000 tokens | ~7,500 tokens | 90% |
| IP address list | ~50,000 tokens | ~5,000 tokens | 90% |

### API Call Efficiency

**Generic Tool Advantage**:
- Single tool definition in LLM context (vs 100+ specialized tools)
- Reduced decision complexity for agent
- Faster tool selection and invocation

## Best Practices

### 1. Always Use Field Filtering
```python
# ❌ Bad: Retrieve all fields
get_objects(object_type="dcim.devices", filters={"site": "main"})

# ✅ Good: Specify needed fields
get_objects(
    object_type="dcim.devices",
    filters={"site": "main"},
    fields=["id", "name", "status", "device_type"]
)
```

### 2. Use Pagination for Large Result Sets
```python
# Query with pagination
get_objects(
    object_type="dcim.devices",
    limit=50,
    offset=0,
    fields=["id", "name"]
)
```

### 3. Filter at API Level, Not in Agent
```python
# ❌ Bad: Fetch all, filter in agent
devices = get_objects(object_type="dcim.devices")
active = [d for d in devices if d.status == "active"]

# ✅ Good: Filter at API level
devices = get_objects(
    object_type="dcim.devices",
    filters={"status": "active"}
)
```

## Advantages Over Specialized Tools

### Token Overhead Comparison

**Specialized Tool Approach** (142+ tools):
- Each tool needs schema in LLM context
- Tool schemas: ~8,000-10,000 tokens
- Agent must choose from 142+ options

**Generic Tool Approach** (3 tools):
- Three tool schemas: ~500-1,000 tokens
- Tool schemas: **90% reduction**
- Agent has simpler decision space

### Maintenance Benefits

- **Single implementation**: One tool handles all object types
- **Easy extension**: New NetBox object types automatically supported
- **Consistent interface**: Same filtering, pagination, field selection across all types
- **Simpler testing**: Test 3 tools, not 142+

## Limitations

### Filter Complexity
Multi-hop filters not supported:
```python
# ❌ Not supported
get_objects(
    object_type="ipam.ip_addresses",
    filters={"device__site__name": "DM-Scranton"}  # Multi-hop
)

# ✅ Workaround: Two-step query
devices = get_objects(
    object_type="dcim.devices",
    filters={"site__name": "DM-Scranton"},
    fields=["id"]
)
ip_addresses = get_objects(
    object_type="ipam.ip_addresses",
    filters={"device_id__in": [d.id for d in devices]}
)
```

### Read-Only Operations
The server does not support:
- Creating new objects
- Updating existing objects
- Deleting objects
- Bulk operations

**Design Rationale**: Read-only interface provides safety for AI agent exploration without risk of accidental infrastructure changes.

## Conclusion

The NetBoxLabs MCP server provides a simple, token-efficient foundation for AI agents to query NetBox infrastructure. Its 3-tool generic design dramatically reduces complexity while maintaining full access to NetBox data through field filtering and flexible queries.

This shared infrastructure enables both Phase 4 agent frameworks (Deepagents and Claude SDK) to focus on orchestration strategies rather than tool implementation, demonstrating that the choice of agent framework is independent of the underlying MCP infrastructure.

---

**Next Steps**: See [Phase 4 Framework Comparison](../phase-4-framework-comparison/overview.md) for how different agent frameworks build upon this shared infrastructure.
