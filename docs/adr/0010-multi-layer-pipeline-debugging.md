# ADR-0010 — Multi-Layer Pipeline Debugging Methodology

## Context

The timeline query failure investigation in tmdbGPT Phase 1 revealed the complexity of debugging multi-stage processing pipelines where failures can cascade across multiple architectural layers. The specific challenge involved:

- **Multiple processing stages**: LLM extraction → Entity resolution → Constraint planning → API execution → Discovery filtering → Step runner filtering → Response formatting → Timeline rendering
- **Hidden failure points**: Initial symptoms ("No summary available") masked the actual failure location deep in constraint validation
- **Layer interdependencies**: Failures in early stages (filtering) manifested as symptoms in later stages (rendering)
- **Data structure mismatches**: Expected vs actual data formats varied between processing layers

Traditional debugging approaches proved insufficient for systematic root cause identification in such complex pipeline architectures.

## Decision

Establish **multi-layer pipeline debugging methodology** using progressive isolation techniques and comprehensive state inspection across architectural boundaries.

**Core Methodology**:
1. **Symptom-to-Source Tracing**: Work backward from user-visible symptoms to identify actual failure points
2. **Layer-by-Layer Isolation**: Create focused debug scripts for each processing stage
3. **State Structure Inspection**: Verify data format assumptions at each layer boundary
4. **Comparative Analysis**: Compare working vs failing query execution paths
5. **Monkey-Patch Diagnostics**: Use non-invasive tracing to capture execution context

**Technical Framework**:
```python
# Stage 1: Symptom Analysis - Timeline Renderer Debug
def debug_timeline_renderer(state):
    print(f"Timeline renderer received:")
    print(f"  state.responses type: {type(state.responses)}")
    print(f"  state.responses length: {len(state.responses)}")
    
    movie_summaries = 0
    for item in state.responses:
        if isinstance(item, dict) and item.get("type") == "movie_summary":
            movie_summaries += 1
    
    print(f"  Movie summaries found: {movie_summaries}")

# Stage 2: Data Flow Analysis - Step Runner Debug  
def debug_step_runner_finalize(self, state):
    print(f"Step runner finalize:")
    print(f"  Input responses: {len(state.responses)}")
    
    # Call original processing
    result = original_finalize(self, state)
    
    print(f"  Output responses: {len(state.responses)}")
    print(f"  Formatted result: {len(state.formatted_response.get('entries', []))}")
    return result

# Stage 3: Constraint Validation Analysis
def debug_filter_validation(entities, constraint_tree, registry):
    print(f"Constraint validation:")
    print(f"  Input entities: {len(entities)}")
    print(f"  Constraint tree: {constraint_tree}")
    
    result = original_filter(entities, constraint_tree, registry)
    print(f"  Filtered result: {len(result)}")
    return result
```

**Progressive Debugging Approach**:
1. **End-to-End Execution**: Run full query and capture final symptoms
2. **Renderer Layer**: Verify renderer receives expected data structure
3. **Processing Layer**: Check data transformation and filtering stages
4. **Constraint Layer**: Validate constraint evaluation and filtering logic
5. **Data Layer**: Verify API data extraction and score assignment

## Consequences

### Positive
- **Systematic Root Cause Identification**: Methodology successfully identified dual-layer constraint validation failure
- **Reusable Diagnostic Framework**: Creates proven approach for future complex pipeline debugging
- **Non-Invasive Analysis**: Monkey patching preserves production code while enabling deep inspection
- **Layer-Boundary Clarity**: Clearly isolates failures to specific architectural layers
- **Comparative Debugging**: Enables systematic comparison of working vs failing execution paths

### Negative
- **Debugging Complexity**: Requires understanding of complete pipeline architecture
- **Multiple Debug Scripts**: Need separate diagnostic tools for each layer
- **Investigation Time**: Systematic approach takes longer than ad-hoc debugging
- **Technical Depth Required**: Demands deep familiarity with internal data structures and flow

### Implementation Benefits
- **Precise Problem Location**: Timeline query issue isolated to constraint validation layers (not rendering)
- **Data Structure Validation**: Confirmed `state.responses` empty vs renderer expecting populated list
- **Execution Flow Mapping**: Traced data flow from 792 summaries → 0 summaries → empty timeline
- **Solution Validation**: Verified fix resolved issue without affecting other layers

## Compliance

This methodology aligns with established debugging practices:
- **Systematic Analysis**: Follows structured debugging approach from software reliability engineering
- **Observability Patterns**: Adapts distributed systems observability for single-process pipelines
- **Root Cause Analysis**: Implements methodical failure analysis as recommended in SRE practices
- **Documentation**: Preserves institutional knowledge for future debugging scenarios

## Related Decisions

- Extends ADR-0003 diagnostic-first methodology with specific multi-layer pipeline techniques
- Supports ADR-0009 endpoint-aware validation by providing the debugging methodology that identified the dual-layer filtering issue
- Establishes foundation for future complex pipeline debugging in tmdbGPT architecture

## Methodology Validation

**Debug Progression Example**:
```
Timeline Query: "First movies by Steven Spielberg"

Stage 1 - Symptom: Timeline renderer returns 0 entries
  ❌ Expected: Chronologically sorted movie list
  ✅ Confirmed: Renderer logic correct, empty input data

Stage 2 - Data Flow: Step runner finalize
  ❌ 792 responses → 0 responses (massive filtering)
  ✅ Identified: Constraint validation removing all results

Stage 3 - Constraint Analysis: filter_valid_movies_or_tv
  ❌ Person credit summaries fail constraint validation
  ✅ Root cause: Backwards validation logic for person endpoints

Stage 4 - Solution Implementation: Endpoint-aware bypass
  ✅ Single-person queries bypass constraint validation
  ✅ Multi-constraint queries preserve validation logic
  ✅ Timeline queries return 332 properly sorted entries
```

This systematic approach successfully identified the precise failure point and enabled targeted solution implementation without affecting unrelated system components.