# Phase 2 NetBox Demo Scenarios

## Production-Ready Demo Scenarios

### Demo 1: VLAN Discovery with Pagination

**User Query**: "List all VLANs in the network"

**Processing Flow**:
1. **Intent Recognition**: Identifies bulk VLAN discovery request
2. **Pagination Strategy**: Implements smart pagination to avoid token overflow
3. **Summary Generation**: Provides executive summary with drill-down options

**Command Execution**:
```python
# MCP tool invocation
result = list_vlans(limit=50, include_summary=True)
```

**Expected Output**:
```
VLAN Summary (Corporate Network)
================================
Total VLANs: 127 across 8 sites

Top VLANs by Size:
- VLAN 100 (Production): 245 active IPs / 512 capacity (47.8% utilization)
- VLAN 200 (Development): 89 active IPs / 256 capacity (34.7% utilization)  
- VLAN 300 (Management): 34 active IPs / 64 capacity (53.1% utilization)

Site Distribution:
- HQ-DC1: 45 VLANs
- Branch-NYC: 23 VLANs
- Branch-LAX: 18 VLANs
- (5 additional sites...)

Showing first 10 VLANs (use 'show more VLANs' for additional results):
1. VLAN 100 - Production - 192.168.100.0/23 - 245 devices
2. VLAN 101 - Production-DMZ - 192.168.102.0/24 - 12 devices
3. VLAN 200 - Development - 172.16.200.0/24 - 89 devices
...
```

**Follow-up Scenarios**:
- "Show more VLANs" → Next pagination batch
- "Details for VLAN 100" → Deep dive into specific VLAN
- "Show VLAN utilization chart" → Performance analysis

---

### Demo 2: Device Interface Analysis

**User Query**: "Show interfaces for device SW-CORE-01"

**Processing Flow**:
1. **Device Resolution**: Resolves device name to NetBox device ID
2. **Interface Enumeration**: Fetches all interfaces with current status
3. **Relationship Mapping**: Includes connected devices, VLANs, IP assignments
4. **Structured Presentation**: Organizes data for clear comprehension

**Command Execution**:
```python
# Multi-tool orchestration
device = get_device_by_id(device_name="SW-CORE-01")
interfaces = list_interfaces(device_id=device.id, include_connections=True)
vlans = list_vlan_assignments(device_id=device.id)
```

**Expected Output**:
```
Device: SW-CORE-01 (Cisco Catalyst 9500-48Y4C)
Location: HQ-DC1, Rack A-01, Position 42
Status: Active, Last seen: 2 minutes ago

Interface Summary:
==================
Total Interfaces: 52 (48 copper + 4 fiber)
Active: 47 (90.4% utilization)
Connected: 44 (84.6% with established links)

Physical Interfaces:
-------------------
GigabitEthernet1/0/1    | UP/UP     | Access VLAN 100    | Connected to: WS-FL1-001
GigabitEthernet1/0/2    | UP/UP     | Access VLAN 100    | Connected to: WS-FL1-002  
GigabitEthernet1/0/3    | DOWN/DOWN | Unassigned          | Unused
...
GigabitEthernet1/0/24   | UP/UP     | Trunk VLANs 100,200,300 | Connected to: SW-ACCESS-01

Fiber Uplinks:
-------------  
TenGigE1/1/1           | UP/UP     | Trunk All VLANs     | Connected to: RTR-CORE-01
TenGigE1/1/2           | UP/UP     | Trunk All VLANs     | Connected to: RTR-CORE-02
TenGigE1/1/3           | UP/UP     | Backup Link         | Connected to: SW-CORE-02
TenGigE1/1/4           | DOWN/DOWN | Spare               | Unused

VLAN Assignments:
================
VLAN 100: 24 interfaces (Production network)
VLAN 200: 8 interfaces (Development network)  
VLAN 300: 4 interfaces (Management network)
Trunk ports: 4 interfaces (carrying all VLANs)

Connection Health:
=================
✓ All critical uplinks operational
✓ No interface errors in last 24 hours
⚠ 3 unused interfaces available for expansion
ℹ Last configuration change: 5 days ago
```

**Follow-up Scenarios**:
- "Show cable details for port 1/0/1" → Physical connection tracing
- "Check interface utilization trends" → Performance analysis
- "Show all connected workstations" → Connected device discovery

---

### Demo 3: Cable Relationship Mapping

**User Query**: "Explain cable relationships for device RTR-CORE-01"

**Processing Flow**:
1. **Device Identification**: Resolves router device in NetBox
2. **Cable Discovery**: Finds all cable connections (copper, fiber, power)
3. **Path Tracing**: Maps end-to-end connectivity chains
4. **Relationship Visualization**: Creates structured relationship map

**Command Execution**:
```python
# Complex relationship traversal
device = get_device_by_id(device_name="RTR-CORE-01")
cables = list_cables(device_id=device.id, include_paths=True)
power_connections = list_power_connections(device_id=device.id)
cable_paths = trace_cable_paths(device.id)
```

**Expected Output**:
```
Cable Relationship Map: RTR-CORE-01
===================================

Data Connections:
================

Primary Uplinks (Redundant):
┌─ RTR-CORE-01 [Te0/0/0] ──[Fiber/SM]──→ RTR-ISP-01 [Gi0/0/1] (Primary ISP)
└─ RTR-CORE-01 [Te0/0/1] ──[Fiber/SM]──→ RTR-ISP-02 [Gi0/0/1] (Backup ISP)

Core Switch Connections:
┌─ RTR-CORE-01 [Te0/1/0] ──[Fiber/MM]──→ SW-CORE-01 [Te1/1/1] 
├─ RTR-CORE-01 [Te0/1/1] ──[Fiber/MM]──→ SW-CORE-02 [Te1/1/1]
└─ RTR-CORE-01 [Te0/1/2] ──[Fiber/MM]──→ SW-DIST-01 [Te2/1/1]

Backup/Management:
┌─ RTR-CORE-01 [Fa0/0] ──[Copper/Cat6]──→ SW-MGMT-01 [Gi0/15] (OOB Mgmt)
└─ RTR-CORE-01 [Console] ──[Serial]────→ TS-CONSOLE-01 [Port8] (Console)

Power Connections:
=================
Primary Power:
┌─ RTR-CORE-01 [PSU-1] ──[C13]──→ PDU-A-01 [Outlet-12] ← UPS-A-01
└─ RTR-CORE-01 [PSU-2] ──[C13]──→ PDU-B-01 [Outlet-12] ← UPS-B-01

Cable Health Analysis:
=====================
✓ All critical paths have redundancy
✓ No single points of failure detected  
✓ Power feeds from separate UPS systems
⚠ Console connection shows intermittent issues (3 incidents last month)
ℹ Fiber optic links tested: 2 days ago (all within spec)

Path Redundancy Matrix:
======================
Internet Access: ✓✓ (Dual ISP, different providers)
Core Switching: ✓✓✓ (Three independent paths)  
Management Access: ✓✓ (Primary + OOB + Console)
Power Supply: ✓✓ (Dual feed, separate UPS)

Cable Inventory:
===============
- 2× Single-mode fiber (ISP uplinks) - 1500m total
- 5× Multi-mode fiber (internal) - 850m total  
- 2× Cat6 Copper (management) - 15m total
- 1× Serial console cable - 3m
- 2× C13 Power cables - 2m each
```

**Follow-up Scenarios**:
- "Show fiber test results" → Cable performance data
- "Trace path to internet" → End-to-end connectivity analysis  
- "Check power redundancy" → Power infrastructure analysis

---

### Demo 4: Complex Infrastructure Audit  

**User Query**: "Perform complete infrastructure audit for site HQ-DC1"

**Processing Flow**:
1. **Site Scope Definition**: Identifies all resources within site boundary
2. **Multi-Domain Analysis**: Covers devices, power, cooling, connectivity
3. **Health Assessment**: Evaluates current state vs. best practices
4. **Optimization Recommendations**: Suggests improvements and expansions

**Expected Output Summary**:
```
Infrastructure Audit Report: HQ-DC1
===================================
Audit Date: 2025-08-10 14:35 UTC
Scope: Complete site infrastructure analysis

Executive Summary:
- Total Devices: 127 (124 active, 3 maintenance)
- Rack Utilization: 67% (32 of 48 racks occupied)  
- Power Utilization: 78% (234kW used / 300kW capacity)
- Network Utilization: 45% average across all links
- Identified Issues: 12 minor, 3 moderate, 0 critical

[Detailed sections follow with specific recommendations...]
```

## Setup Instructions for Demo Environment

### Prerequisites
```bash
# Clone repository
git clone https://github.com/FinnMacCumail/mcp-netbox.git
cd mcp-netbox

# Python environment  
python -m venv netbox-mcp-env
source netbox-mcp-env/bin/activate
pip install -r requirements.txt
```

### NetBox Configuration
```bash
# Environment variables
export NETBOX_URL="https://your-netbox-instance.com"
export NETBOX_TOKEN="your-netbox-api-token"
export MCP_PORT="8000"
```

### MCP Server Startup
```bash
# Start MCP server
python -m mcp_netbox.server --host 0.0.0.0 --port 8000

# Verify server health
curl http://localhost:8000/health
```

### Claude Code Integration  
```bash
# Configure Claude Code MCP connection
claude code --mcp-server http://localhost:8000

# Test connectivity
claude code "List available MCP tools"
```

### Demo Data Requirements

**Minimum NetBox Data for Demos**:
- 10+ devices across 2+ sites
- 50+ interfaces with various states
- 20+ VLANs with IP assignments  
- 25+ cables connecting devices
- Power infrastructure (PDUs, outlets)

**Recommended Test Queries**:
1. "Show me all devices in site HQ-DC1"
2. "List VLANs with their utilization"  
3. "Trace connections for device SW-CORE-01"
4. "Show power distribution for rack A-01"
5. "Find unused interfaces across all devices"

Each demo scenario includes comprehensive logging to demonstrate the internal decision-making process, tool orchestration, and optimization strategies.