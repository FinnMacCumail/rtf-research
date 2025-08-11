# ADR-0003 — Diagnostic-First Debugging Methodology

## Context

Complex multi-stage processing pipelines (like tmdbGPT's query routing system) present unique debugging challenges:
- Failures can occur across multiple modules with interdependent execution flows
- Traditional logging often misses critical execution context at failure points
- Root cause analysis requires tracing execution paths through semantic and symbolic processing stages
- Mixed content issues required understanding the complete query → result transformation pipeline

## Decision

Adopt a **diagnostic-first debugging methodology** using comprehensive execution tracing with monkey patching for systematic root cause analysis in complex pipeline failures.

**Core Approach**:
- Create comprehensive execution tracers that intercept key pipeline functions
- Use monkey patching to trace execution flow without invasive code changes
- Implement comparative analysis between working and failing query patterns
- Document systematic diagnostic procedures for future pipeline debugging

**Technical Implementation**:
```python
def create_fallback_tracer():
    original_functions = {}
    
    def trace_function(module_path, func_name, custom_handler=None):
        # Intercept and log function calls with full context
        # Map execution flow across module boundaries
        pass
    
    # Target critical pipeline stages
    trace_function('core.planner.plan_utils', 'execute_plan')
    trace_function('core.llm.role_inference', 'infer_media_type_from_query')
    trace_function('core.entity.entity_resolution', 'resolve_network')
```

## Consequences

### Positive
- **Precise Failure Identification**: Maps exact failure points in complex multi-module pipelines
- **Systematic Root Cause Analysis**: Enables methodical comparison of working vs. failing execution paths
- **Non-Invasive Debugging**: Monkey patching preserves production code integrity during diagnosis
- **Reusable Diagnostic Patterns**: Creates proven methodology for future pipeline debugging scenarios
- **Complete Execution Visibility**: Captures full context across module boundaries and processing stages

### Negative
- **Additional Complexity**: Requires understanding of internal function signatures and execution flow
- **Maintenance Overhead**: Tracers may need updates when underlying functions change
- **Performance Impact**: Comprehensive tracing adds execution overhead during debugging sessions

## Compliance

This approach aligns with established software engineering practices:
- **Observability**: Follows distributed systems observability patterns adapted for single-process pipelines
- **Systematic Debugging**: Implements structured debugging methodology from software reliability engineering
- **Documentation**: Creates institutional knowledge preservation as recommended in ADR best practices

## Related Decisions

- Builds on ADR-0002 chatbox architecture by providing debugging methodology for the established pipeline
- Supports the hybrid semantic/symbolic approach by enabling diagnosis of both processing paths
- Complements constraint relaxation strategies by identifying where relaxation occurs in the pipeline