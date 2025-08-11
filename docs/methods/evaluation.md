# Evaluation

- **Functional tests**: unit/integration tests for planning, reranking, validation, and formatting
- **Token-boundary tests**: ensure pagination/summarization works within LLM context limits
- **Trace analysis**: execution traces for understanding multi-step plans and fallbacks

## Advanced Diagnostic Methodologies

### Execution Tracing for Complex Pipelines

**Monkey Patching Approach**: For debugging multi-stage processing pipelines, comprehensive tracing using function interception proves highly effective:

```python
# Trace key execution points in query processing
def create_fallback_tracer():
    original_functions = {}
    
    def trace_function(module_path, func_name, custom_handler=None):
        # Intercept and log function calls with context
        pass
    
    # Target critical pipeline stages
    trace_function('core.planner.plan_utils', 'execute_plan')
    trace_function('core.llm.role_inference', 'infer_media_type_from_query')
    trace_function('core.entity.entity_resolution', 'resolve_network')
```

**Benefits**:
- Maps complete execution flow across module boundaries
- Identifies precise failure points in complex pipelines
- Enables systematic root cause analysis without invasive code changes

**Use Cases**:
- Mixed content debugging (media type detection issues)
- Entity resolution pipeline failures
- Constraint validation problems
- Fallback system contamination

### Comparative Query Analysis

**Methodology**: Test functionally similar queries with different outcomes to identify behavioral patterns:

- **Working baseline**: "horror films" → correct movie-only results  
- **Failing case**: "comedy shows" → mixed TV/movie results
- **Analysis**: Trace execution differences to isolate root cause

**Learning**: Systematic comparison of working vs. failing cases often reveals single-point failures that comprehensive testing might miss.

**Related Architecture Decisions**: 
- [ADR-0003: Diagnostic-First Debugging Methodology](../adr/0003-diagnostic-first-debugging.md) - Documents the systematic approach to execution tracing
- [ADR-0004: Media Type Detection Enhancement Strategy](../adr/0004-media-type-detection-enhancement.md) - Example application of comparative analysis methodology
