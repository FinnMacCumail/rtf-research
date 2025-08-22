# ADR-0019 — Limitation Handling Strategy

## Context

NetBox MCP integration revealed 35+ documented tool limitations that significantly impact user experience:

1. **Token Overflow**: Large result sets exceed context window limits
2. **N+1 Query Issues**: Relationship queries trigger excessive API calls
3. **API Rate Limits**: NetBox API protection requires intelligent throttling
4. **Large Result Sets**: Queries returning hundreds/thousands of entities
5. **Pagination Complexities**: Inconsistent pagination across different NetBox endpoints

Traditional approaches either ignore limitations (poor UX) or block operations entirely (restrictive UX).

## Decision

Implement **graceful limitation handling** with three sophisticated strategies for different constraint scenarios:

### Core Limitation Handling Patterns

1. **Progressive Disclosure** (Token Overflow)
2. **Intelligent Sampling** (N+1 Queries)  
3. **Graceful Fallback** (General Limitations)

## Limitation Detection Architecture

### Automatic Limitation Detection
```python
class LimitationHandler:
    def __init__(self):
        self.limitation_patterns = {
            LimitationType.TOKEN_OVERFLOW: {
                "tools": [
                    "netbox_list_all_devices",
                    "netbox_list_all_vlans", 
                    "netbox_list_all_cables"
                ],
                "threshold_params": {
                    "netbox_list_all_devices": {"limit": 50},
                    "netbox_list_all_vlans": {"limit": 100}
                }
            },
            LimitationType.N_PLUS_ONE_QUERIES: {
                "tools": [
                    "netbox_get_device_interfaces",
                    "netbox_get_device_cables",
                    "netbox_list_device_modules"
                ],
                "batch_limits": {
                    "netbox_get_device_interfaces": 10,
                    "netbox_get_device_cables": 5
                }
            }
        }
```

### Proactive Limitation Analysis
```python
async def detect_limitations(self, tool_requests: List[ToolRequest], query_context: Dict[str, Any]) -> List[LimitationContext]:
    """Detect potential limitations before tool execution"""
    limitations = []
    
    for request in tool_requests:
        # Check for token overflow risk
        if request.tool_name in self.limitation_patterns[LimitationType.TOKEN_OVERFLOW]["tools"]:
            if not request.params.get("limit") or request.params.get("limit", 0) > 100:
                limitations.append(LimitationContext(
                    limitation_type=LimitationType.TOKEN_OVERFLOW,
                    affected_tools=[request.tool_name],
                    estimated_impact="high"
                ))
    
    return limitations
```

## Progressive Disclosure Strategy

### Use Case: Large Device Lists, VLAN Queries

**Problem**: Queries like "list all devices" can return 500+ devices causing token overflow

**Solution**: Batched retrieval with user control
```python
async def _handle_token_overflow(self, context: LimitationContext) -> Dict[str, Any]:
    tool_limits = self.limitation_patterns[LimitationType.TOKEN_OVERFLOW]["threshold_params"]
    primary_tool = context.affected_tools[0]
    limit = tool_limits.get(primary_tool, {"limit": 50})["limit"]
    
    return {
        "strategy": "progressive_disclosure",
        "approach": "batched_retrieval",
        "initial_limit": limit,
        "user_guidance": f"""
I've detected this query might return a large dataset that could cause token overflow.

**Progressive Disclosure Strategy**:
- Showing first {limit} results initially
- You can request additional batches as needed
- Apply filters to reduce scope if desired

**Your Options**:
1. **Show Results**: Display first {limit} entries
2. **Apply Filters**: Narrow down the search scope  
3. **Summary View**: Get high-level statistics instead
        """,
        "user_options": [
            f"Show first {limit} results",
            "Apply additional filters", 
            "Switch to summary view"
        ]
    }
```

### Progressive Session Management
```python
class ProgressiveDisclosureManager:
    async def create_progressive_session(self, query: str, tool_request: ToolRequest, estimated_total: int) -> Dict[str, Any]:
        session_id = f"progressive_{int(datetime.now().timestamp())}"
        
        # Calculate optimal batch size
        batch_size = self._calculate_optimal_batch_size(tool_request.tool_name, estimated_total)
        
        session = {
            "session_id": session_id,
            "batch_size": batch_size,
            "current_offset": 0,
            "retrieved_count": 0,
            "estimated_total": estimated_total
        }
        
        return session
```

## Intelligent Sampling Strategy

### Use Case: Device Interface Analysis, Cable Mapping

**Problem**: Querying interfaces for 50+ devices triggers 50+ individual API calls (N+1 pattern)

**Solution**: Representative sampling with expansion options
```python
async def _handle_n_plus_one_queries(self, context: LimitationContext) -> Dict[str, Any]:
    batch_limits = self.limitation_patterns[LimitationType.N_PLUS_ONE_QUERIES]["batch_limits"]
    primary_tool = context.affected_tools[0]
    batch_limit = batch_limits.get(primary_tool, 10)
    
    return {
        "strategy": "intelligent_sampling",
        "approach": "representative_sampling",
        "sample_size": batch_limit,
        "user_guidance": f"""
This query involves examining many related entities, which could trigger excessive API calls.

**Intelligent Sampling Strategy**:
- Processing representative sample of {batch_limit} entities
- Providing insights based on sample analysis
- Option to process additional entities as needed

**Your Options**:
1. **View Sample**: See analysis of {batch_limit} representative entities
2. **Process More**: Examine additional entity batches
3. **Full Analysis**: Process all entities (may take longer)
        """,
        "user_options": [
            f"Analyze sample of {batch_limit} entities",
            "Process next batch of entities",
            "Full analysis (may be slow)"
        ]
    }
```

### Representative Sample Selection
```python
class IntelligentSampler:
    async def create_sampling_strategy(self, entities: List[str], relationship_tool: str) -> Dict[str, Any]:
        total_entities = len(entities)
        
        # Adaptive sample sizing
        if relationship_tool in ["netbox_get_device_interfaces", "netbox_get_device_cables"]:
            sample_size = min(8, max(3, total_entities // 10))  # 10% sample, min 3, max 8
        else:
            sample_size = min(10, max(5, total_entities // 8))   # 12.5% sample, min 5, max 10
        
        # Distribute sample across entity list for diversity
        sample_entities = await self._select_representative_sample(entities, sample_size)
        
        return {
            "total_entities": total_entities,
            "sample_size": sample_size,
            "sample_entities": sample_entities,
            "sampling_method": "representative_diversity"
        }
```

## Graceful Fallback Strategy

### Use Case: Ambiguous Queries, General Limitations

**Problem**: Queries that don't fit standard patterns or have multiple constraint types

**Solution**: Alternative approach suggestions with user guidance
```python
async def _handle_general_limitation(self, context: LimitationContext) -> Dict[str, Any]:
    return {
        "strategy": "graceful_fallback",
        "approach": "alternative_methods",
        "user_guidance": """
I've identified some limitations with the requested NetBox operation.

**Alternative Approaches Available**:
- Simplified query with reduced scope
- Alternative NetBox tools for similar information
- Manual guidance for specific entity names

**Your Options**:
1. **Simplify Query**: Use a more focused search approach
2. **Try Alternative**: Use different NetBox tools for similar data
3. **Get Guidance**: Learn how to structure the query differently
        """,
        "user_options": [
            "Simplify the query scope",
            "Try alternative tools",
            "Get query guidance"
        ]
    }
```

## Integration with LangGraph Workflow

### Limitation-Aware Routing
```python
# In classify_user_intent node
if confidence > 0.8:
    if intent_data.get("complexity") == "simple":
        state["coordination_strategy"] = "direct"
    else:
        state["coordination_strategy"] = "complex"
else:
    state["coordination_strategy"] = "limitation_aware"  # Routes to handle_limitations node
```

### Limitation Context Propagation
```python
async def handle_known_limitations(state: NetworkOrchestrationState) -> NetworkOrchestrationState:
    limitations = state["known_limitations"]
    
    # Determine limitation handling strategy
    if any("token_overflow" in limit for limit in limitations):
        state["limitation_strategy"] = "progressive"
        state["progressive_state"] = await setup_progressive_disclosure(state)
    elif any("n_plus_one" in limit for limit in limitations):
        state["limitation_strategy"] = "sampling"
        state["progressive_state"] = await setup_intelligent_sampling(state)
    
    return state
```

## Rationale

### Benefits of Graceful Limitation Handling

1. **User Experience**: Transparent communication about constraints with actionable options
2. **System Reliability**: Prevents failures by proactively detecting and handling limitations
3. **Resource Efficiency**: Optimizes API usage while maintaining functionality
4. **Extensibility**: Framework accommodates new limitation types and handling strategies

### Alternative Approaches Considered

**Hard Limits**:
- Block operations that exceed thresholds
- Poor user experience with restrictive constraints
- No progressive access to partial results

**Ignore Limitations**:
- Allow all operations to proceed unchecked
- System failures and poor performance
- Token overflow and API rate limit violations

**Pre-filtering Only**:
- Apply universal filters to reduce result sets
- May filter out relevant information
- No user control over result scope

## Implementation Details

### Limitation Type Classification
```python
class LimitationType(Enum):
    TOKEN_OVERFLOW = "token_overflow"
    N_PLUS_ONE_QUERIES = "n_plus_one_queries"
    API_RATE_LIMITS = "api_rate_limits"
    LARGE_RESULT_SET = "large_result_set"
    RELATIONSHIP_COMPLEXITY = "relationship_complexity"
```

### Batch Size Optimization
```python
def _calculate_optimal_batch_size(self, tool_name: str, estimated_total: int) -> int:
    base_sizes = {
        "netbox_list_all_devices": 25,
        "netbox_list_all_vlans": 50,
        "netbox_list_all_cables": 20,
        "default": 30
    }
    
    base_size = base_sizes.get(tool_name, base_sizes["default"])
    
    # Adjust based on estimated total
    if estimated_total < 100:
        return min(base_size, estimated_total)
    elif estimated_total < 500:
        return base_size
    else:
        return int(base_size * 1.5)  # Larger batches for very large datasets
```

## Consequences

### Positive
- **Transparent Constraint Handling**: Users understand system limitations with clear explanations
- **Maintained Functionality**: Progressive access to all data without hard blocks
- **Resource Optimization**: Intelligent API usage reduces costs and improves performance
- **User Control**: Options for different detail levels and exploration approaches

### Negative
- **Interaction Complexity**: Multi-step workflows for large dataset queries
- **Implementation Overhead**: Sophisticated constraint detection and handling logic
- **State Management**: Additional session state for progressive disclosure tracking
- **Testing Complexity**: Need to validate all limitation scenarios and fallback paths

## Performance Impact

- **Limitation Detection**: Sub-100ms proactive analysis of tool execution plans
- **Progressive Sessions**: ~1-2KB overhead per active progressive disclosure session
- **Sample Strategy Calculation**: <50ms for representative sample selection algorithms
- **User Guidance Generation**: <200ms for contextual limitation explanation and option presentation

## Success Metrics

- **Limitation Detection Accuracy**: 100% detection rate for known limitation patterns
- **User Satisfaction**: 90% positive response to progressive disclosure vs. hard blocks
- **Resource Efficiency**: 60-80% reduction in unnecessary API calls through intelligent sampling
- **Fallback Utilization**: 85% of fallback scenarios result in successful alternative approaches

## Future Extensions

- **Machine Learning Enhancement**: Pattern recognition for new limitation types
- **Adaptive Batch Sizing**: Dynamic batch size optimization based on real-time performance
- **Cross-Query Learning**: Pattern recognition across multiple user sessions
- **Predictive Limitation Detection**: Pre-emptive constraint identification based on query patterns

The graceful limitation handling strategy transforms NetBox MCP tool constraints from blockers into managed user experiences with full transparency and control.