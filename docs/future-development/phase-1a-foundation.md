# Phase 3: OpenAI Agent Orchestration Foundation

## Objective

Replace unreliable Claude CLI process management with efficient query orchestration using **OpenAI GPT-4o-mini** for intent parsing while providing graceful coordination of existing NetBox MCP tools and establishing a solid foundation for advanced features.

## Critical Limitation Handling

### Graceful Token Overflow Management

**Problem**: Large NetBox responses exceed Claude's context window, causing query failures.

**Current Failures**:
- VLAN listings with >100 VLANs: 100% failure rate
- Device interface queries for core switches: 85% failure rate  
- Site-wide infrastructure queries: 90% failure rate

**Orchestration Strategy**:
```python
def implement_intelligent_limitation_handling(query_type, estimated_size):
    """
    Graceful handling of tool limitations through orchestration
    """
    if estimated_size > TOKEN_THRESHOLD:
        return {
            'strategy': 'orchestrated_summary',
            'coordination': 'paginated_with_intelligent_aggregation',
            'user_experience': 'progressive_disclosure',
            'fallback': 'simplified_view_with_drill_down'
        }
    else:
        return {'strategy': 'direct_tool_response'}

# Example orchestration for VLAN queries
def coordinate_vlan_listing_with_limitations(filters=None):
    # Work with existing tool as-is, provide intelligent coordination
    first_batch = existing_vlan_tool(limit=25, offset=0, filters=filters)
    total_estimate = estimate_from_first_batch(first_batch)
    
    return {
        'type': 'orchestrated_response',
        'summary': generate_intelligent_summary(first_batch, total_estimate),
        'sample_data': first_batch,
        'coordination_options': [
            'Show more VLANs (next 25)',
            'Filter by specific criteria',
            'Switch to summary view'
        ]
    }
```

**Coordination Strategy**:
- Work with existing tool limitations, provide graceful user experience
- Implement intelligent aggregation and summarization
- Validate orchestrated responses provide value within constraints

### Intelligent N+1 Query Coordination

**Problem**: Individual relationship lookups create excessive API call patterns with existing tools.

**Orchestration Case Study - VLAN Query Coordination**:
```python
# CURRENT: N+1 Pattern (127 API calls for 63 VLANs using existing tools)
def coordinate_vlan_details_with_existing_tools(vlan_list):
    # Work with existing tools as-is, provide intelligent coordination
    results = []
    for vlan in vlan_list:
        vlan_data = existing_vlan_tool(vlan.id)           # Current tool limitation
        devices = existing_device_tool(vlan_filter=vlan.id)  # Work with what's available
        interfaces = existing_interface_tool(vlan_filter=vlan.id)  # Use existing capabilities
        results.append({
            'vlan': vlan_data,
            'devices': devices, 
            'interfaces': interfaces
        })
    return results

# ORCHESTRATED: Intelligent Coordination (work with tools as-is)
def orchestrate_vlan_details_gracefully(vlan_list):
    # Accept tool limitations, provide intelligent user experience
    if len(vlan_list) > 10:  # Prevent overwhelming API calls
        return {
            'strategy': 'progressive_loading',
            'initial_sample': coordinate_sample_vlans(vlan_list[:5]),
            'remaining_count': len(vlan_list) - 5,
            'user_options': [
                'Load next 5 VLANs',
                'Search specific VLAN',
                'Switch to summary view'
            ]
        }
    else:
        return coordinate_vlan_details_with_existing_tools(vlan_list)
```

**Coordination Impact**:
- User experience: Graceful handling of tool limitations
- Performance: Intelligent load management with existing tools

## OpenAI-Powered Query Classification System

### Intent Recognition Using GPT-4o-mini

**Architecture**:
```python
class QueryIntentClassifier:
    def __init__(self):
        self.client = OpenAI()
        self.intent_patterns = {
            'device_lookup': 'Single device information request',
            'bulk_discovery': 'List/search multiple objects',
            'relationship_analysis': 'Connection/dependency mapping',
            'performance_query': 'Utilization/metrics analysis',
            'configuration_audit': 'Compliance/standard checking'
        }
    
    async def classify_intent(self, user_query: str) -> Dict[str, Any]:
        """
        Use GPT-4o-mini to understand user query intent and extract parameters
        """
        classification_prompt = f"""
        Analyze this NetBox query and identify:
        1. Primary intent (device_lookup, bulk_discovery, relationship_analysis, etc.)
        2. Target objects (devices, VLANs, interfaces, cables, etc.)  
        3. Specific filters or constraints
        4. Expected response format
        5. Complexity level (simple, moderate, complex)
        
        Query: "{user_query}"
        
        Respond in JSON format.
        """
        
        response = await self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": classification_prompt}],
            temperature=0.1  # Low temperature for consistent classification
        )
        
        return json.loads(response.choices[0].message.content)
```

**Query Routing Logic**:
```python
async def route_query(classified_intent: Dict[str, Any]) -> ExecutionPlan:
    """
    Create execution plan based on classified intent
    """
    complexity = classified_intent['complexity_level']
    target_objects = classified_intent['target_objects']
    
    if complexity == 'simple' and len(target_objects) == 1:
        return create_direct_api_plan(classified_intent)
    elif complexity == 'moderate' and 'relationship' in classified_intent['intent']:
        return create_multi_step_plan(classified_intent)
    elif complexity == 'complex':
        return create_orchestrated_plan(classified_intent)
    else:
        return create_fallback_plan(classified_intent)

def create_direct_api_plan(intent: Dict[str, Any]) -> ExecutionPlan:
    """Simple, single-tool execution"""
    return ExecutionPlan(
        steps=[select_appropriate_mcp_tool(intent)],
        estimated_time=1,
        fallback_strategy='retry_with_relaxed_filters'
    )

def create_multi_step_plan(intent: Dict[str, Any]) -> ExecutionPlan:
    """Multi-step execution with dependency handling"""
    steps = []
    if 'device' in intent['target_objects']:
        steps.append(resolve_device_references(intent))
    if 'relationship' in intent['intent']:
        steps.append(fetch_relationships(intent))  
    steps.append(aggregate_and_format(intent))
    
    return ExecutionPlan(
        steps=steps,
        estimated_time=3,
        fallback_strategy='progressive_simplification'
    )
```

## LangGraph Tool Orchestration Engine

### Query Pattern Recognition

**Identified Common Patterns**:
```python
class NetworkQueryPatterns:
    DEVICE_TO_INTERFACES = "device → interfaces → connection details"
    SITE_TO_UTILIZATION = "site → racks → devices → utilization metrics"
    VLAN_TO_TOPOLOGY = "VLAN → interfaces → devices → physical topology"
    CABLE_TO_PATH = "cable → endpoints → device relationships → network path"
    IP_TO_ALLOCATION = "IP range → assignments → device mappings → usage analysis"
```

### LangGraph State Machine Implementation

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class NetworkQueryState(TypedDict):
    user_query: str
    classified_intent: Dict[str, Any]
    intermediate_results: List[Dict[str, Any]]
    final_response: Optional[str]
    limitation_state: Optional[str]  # Track known tool limitations
    retry_count: int

def create_network_orchestration_graph():
    """
    Build LangGraph state machine for coordinating existing NetBox MCP tools
    """
    graph = StateGraph(NetworkQueryState)
    
    # Define nodes for tool coordination
    graph.add_node("classify_intent", classify_query_intent)
    graph.add_node("plan_coordination", create_coordination_plan) 
    graph.add_node("execute_simple", coordinate_simple_query)
    graph.add_node("execute_complex", coordinate_complex_tools)
    graph.add_node("handle_limitations", handle_tool_limitations)
    graph.add_node("format_response", format_coordinated_response)
    
    # Define edges and routing logic
    graph.set_entry_point("classify_intent")
    
    graph.add_conditional_edges(
        "classify_intent",
        route_based_on_complexity,
        {
            "simple": "execute_simple",
            "complex": "plan_coordination", 
            "limitation": "handle_limitations"
        }
    )
    
    graph.add_conditional_edges(
        "execute_simple",
        check_coordination_success,
        {
            "success": "format_response",
            "limitation": "handle_limitations"
        }
    )
    
    graph.add_edge("plan_coordination", "execute_complex")
    graph.add_conditional_edges(
        "execute_complex", 
        check_complex_coordination,
        {
            "success": "format_response",
            "partial": "format_response",  # Handle partial results gracefully
            "limitation": "handle_limitations"
        }
    )
    
    graph.add_conditional_edges(
        "handle_limitations",
        decide_fallback_strategy,
        {
            "retry_simplified": "classify_intent",
            "graceful_response": "format_response", 
            "user_guidance": END
        }
    )
    
    graph.add_edge("format_response", END)
    
    return graph.compile()

# Node implementations for coordination
async def classify_query_intent(state: NetworkQueryState) -> NetworkQueryState:
    classifier = QueryIntentClassifier()
    intent = await classifier.classify_intent(state["user_query"])
    
    return {
        **state,
        "classified_intent": intent
    }

async def coordinate_complex_tools(state: NetworkQueryState) -> NetworkQueryState:
    """
    Coordinate multiple existing tools with graceful limitation handling
    """
    plan = state["classified_intent"]["coordination_plan"]
    results = []
    
    for step in plan["steps"]:
        try:
            # Use existing MCP tools as-is
            step_result = await coordinate_existing_tool(step)
            results.append(step_result)
            
            # Pass results to next step if needed
            if step.get("depends_on_previous"):
                step["context"] = results[-1]
                
        except KnownToolLimitation as e:
            return {
                **state,
                "limitation_state": f"Tool limitation: {step['tool']} - {str(e)}",
                "intermediate_results": results
            }
    
    return {
        **state,
        "intermediate_results": results
    }
```

## Performance & Reliability Optimization

### Efficient API Communication

**Connection Pooling Implementation**:
```python
import aiohttp
import asyncio
from aiohttp_retry import RetryClient, ExponentialRetry

class OptimizedNetBoxClient:
    def __init__(self, base_url: str, token: str):
        self.base_url = base_url
        self.headers = {
            'Authorization': f'Token {token}',
            'Content-Type': 'application/json'
        }
        
        # Connection pool configuration
        connector = aiohttp.TCPConnector(
            limit=20,  # Total connection pool size
            limit_per_host=10,  # Connections per host
            keepalive_timeout=30,
            enable_cleanup_closed=True
        )
        
        # Retry strategy for transient failures
        retry_options = ExponentialRetry(
            attempts=3,
            start_timeout=1,
            max_timeout=10,
            statuses={500, 502, 503, 504, 429}  # Retry on these HTTP codes
        )
        
        self.session = RetryClient(
            connector=connector,
            retry_options=retry_options,
            timeout=aiohttp.ClientTimeout(total=30)
        )
    
    async def parallel_requests(self, endpoints: List[str]) -> List[Dict]:
        """
        Execute multiple API requests concurrently
        """
        tasks = [
            self.session.get(f"{self.base_url}{endpoint}", headers=self.headers)
            for endpoint in endpoints
        ]
        
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        
        results = []
        for response in responses:
            if isinstance(response, Exception):
                results.append({"error": str(response)})
            else:
                results.append(await response.json())
                
        return results
```

### Intelligent Caching Strategy

```python
import redis
import json
import hashlib
from typing import Optional, Any

class NetworkQueryCache:
    def __init__(self, redis_url: str):
        self.redis_client = redis.from_url(redis_url)
        self.default_ttl = 300  # 5 minutes
        
        # Different TTLs based on data volatility
        self.ttl_by_type = {
            'device_info': 3600,      # 1 hour (relatively static)
            'interface_status': 300,   # 5 minutes (changes frequently)
            'vlan_assignments': 1800,  # 30 minutes (moderate changes)
            'cable_connections': 7200, # 2 hours (very static)
            'utilization_data': 60     # 1 minute (highly dynamic)
        }
    
    def generate_cache_key(self, query_type: str, parameters: Dict[str, Any]) -> str:
        """
        Generate consistent cache key from query parameters
        """
        # Sort parameters for consistent key generation
        param_str = json.dumps(parameters, sort_keys=True)
        param_hash = hashlib.md5(param_str.encode()).hexdigest()
        
        return f"netbox:{query_type}:{param_hash}"
    
    async def get_cached_result(self, query_type: str, parameters: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """
        Retrieve cached query result if available
        """
        cache_key = self.generate_cache_key(query_type, parameters)
        cached_data = self.redis_client.get(cache_key)
        
        if cached_data:
            return json.loads(cached_data)
        
        return None
    
    async def cache_result(self, query_type: str, parameters: Dict[str, Any], result: Dict[str, Any]):
        """
        Cache query result with appropriate TTL
        """
        cache_key = self.generate_cache_key(query_type, parameters)
        ttl = self.ttl_by_type.get(query_type, self.default_ttl)
        
        self.redis_client.setex(
            cache_key,
            ttl,
            json.dumps(result, default=str)  # Handle datetime serialization
        )
```

## User Experience Improvements

### Progress Indicators & Streaming

```python
import asyncio
from typing import AsyncGenerator

async def execute_with_progress(query_plan: ExecutionPlan, 
                               websocket_connection) -> AsyncGenerator[str, None]:
    """
    Execute query with real-time progress updates via WebSocket
    """
    total_steps = len(query_plan.steps)
    
    yield f"Starting execution: {total_steps} steps planned"
    
    for i, step in enumerate(query_plan.steps, 1):
        yield f"Step {i}/{total_steps}: {step.description}"
        
        try:
            step_result = await execute_step_with_timeout(step, timeout=30)
            yield f"Step {i} completed: {step.success_message}"
            
            # Send intermediate results if valuable
            if step.stream_intermediate_results:
                yield f"Partial results: {format_intermediate_result(step_result)}"
                
        except asyncio.TimeoutError:
            yield f"Step {i} timed out, falling back to simplified approach"
            step_result = await execute_simplified_fallback(step)
        except Exception as e:
            yield f"Step {i} failed: {str(e)}, attempting recovery"
            step_result = await execute_step_recovery(step, e)
    
    yield "Execution completed, formatting final response"
```

### Natural Language Error Explanations

```python
async def generate_user_friendly_error(error: Exception, 
                                     context: Dict[str, Any]) -> str:
    """
    Convert technical errors to actionable user messages using OpenAI
    """
    error_context_prompt = f"""
    Convert this technical error into a helpful explanation for a network engineer:
    
    Error: {str(error)}
    Context: {json.dumps(context, indent=2)}
    
    Provide:
    1. What went wrong in simple terms
    2. Likely causes  
    3. Suggested next steps
    4. Alternative queries they could try
    
    Be concise and actionable.
    """
    
    client = OpenAI()
    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": error_context_prompt}],
        temperature=0.3
    )
    
    return response.choices[0].message.content
```

## Success Metrics

### Performance Targets
- **Simple queries**: Complete in under 2 seconds (vs current 3-10 seconds)
- **Complex multi-step queries**: Complete in under 30 seconds (vs current 3+ minutes)  
- **Query success rate**: 100% without requiring multiple attempts
- **Token overflow errors**: Zero occurrences

### Cost Efficiency
- **Per-query cost reduction**: 90% reduction through Claude CLI elimination
- **API call efficiency**: 80-95% reduction in total API calls through optimization
- **Infrastructure costs**: Minimal additional overhead

### User Experience
- **Response relevance**: 95% of queries return actionable information
- **Error rate**: <5% of queries encounter unrecoverable errors
- **Learning curve**: New users productive within 1 day instead of 1 week

This foundation phase establishes the reliability and performance baseline required for the advanced capabilities planned in subsequent phases.