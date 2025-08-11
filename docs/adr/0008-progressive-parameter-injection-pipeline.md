# ADR-0008 — Progressive Parameter Injection Pipeline

## Context

Complex natural language queries require systematic parameter injection across multiple processing stages to transform user intent into executable TMDB API calls. The tmdbGPT system handles diverse constraint types (entities, dates, revenue, ratings) that must be progressively injected and validated through the pipeline:

**Challenge Areas**:
- **Entity-to-Parameter Mapping**: Converting extracted entities to appropriate TMDB API parameters
- **Constraint System Integration**: Injecting constraint-derived parameters alongside entity-based parameters
- **Semantic Parameter Inference**: Adding contextual parameters not explicitly mentioned in queries
- **Parameter Conflict Resolution**: Handling overlapping or contradictory parameter sources
- **Validation and Fallback**: Ensuring parameter combinations produce valid API calls

**Example Query Complexity**:
"2010s horror movies with revenue under $25M, highly rated"
- **Entities**: genre=horror, date=2010s, revenue=25000000
- **Constraints**: primary_release_date.gte/lte, sort_by, revenue threshold
- **Semantic**: vote_count.gte (inferred from "highly rated")
- **Validation**: Ensure date ranges don't conflict with single year parameters

## Decision

Implement a **four-phase progressive parameter injection pipeline** that systematically builds API parameters from multiple sources with clear precedence rules:

### Phase 1: Entity-Based Parameter Injection
**Source**: Resolved query entities from LLM extraction  
**Purpose**: Core parameter mapping from explicit user mentions  
```python
# core/embeddings/hybrid_retrieval.py - convert_matches_to_execution_steps()
for entity_key, param_name in JOIN_PARAM_MAP.items():
    if entity_key in resolved_entities:
        parameters[param_name] = ",".join(map(str, ids))
```

### Phase 2: Constraint-Based Parameter Override
**Source**: Constraint tree built from query entities  
**Purpose**: Specialized parameter injection with constraint-specific logic  
**Precedence**: Overrides entity-based parameters
```python  
# app.py - plan() function
for constraint in state.constraint_tree.flatten():
    constraint_key = constraint.key
    constraint_value = constraint.value
    step["parameters"][constraint_key] = str(constraint_value)
    
# Handle parameter conflicts (date ranges override single years)
if has_date_range:
    step["parameters"].pop('primary_release_year', None)
```

### Phase 3: Semantic Parameter Inference  
**Source**: Natural language analysis of query text  
**Purpose**: Add contextual parameters not explicitly mentioned  
**Precedence**: Only adds missing parameters, never overrides existing ones
```python
# app.py - plan() function  
optional_params = PlanValidator().infer_semantic_parameters(state.input)
for param_name in optional_params:
    if param_name in SAFE_OPTIONAL_PARAMS and param_name not in step["parameters"]:
        # Contextual parameter injection with query-specific values
        if param_name == "sort_by" and "highest-rated" in query:
            step["parameters"][param_name] = "vote_average.desc"
        elif param_name == "vote_count.gte" and "rated" in query:
            step["parameters"][param_name] = "50"
```

### Phase 4: Revenue Constraint Specialized Injection
**Source**: Revenue-specific constraint mapping  
**Purpose**: Handle financial constraints requiring strategic sorting + post-filtering  
**Integration Point**: Within constraint-based phase with specialized logic
```python
# core/planner/plan_validator.py - inject_parameters_from_query_entities()
if ent_type == "revenue" and resolved_param == "sort_by":
    operator = ent.get("operator", "less_than")
    if operator in ("less_than", "less_than_equal"):
        value = "popularity.desc"  # Strategic sorting for revenue constraints
    elif operator in ("greater_than", "greater_than_equal"):  
        value = "revenue.desc"
```

## Parameter Precedence Hierarchy

1. **Constraint-Based Parameters** (Highest Priority)
   - Date ranges, revenue thresholds, rating constraints
   - Derived from systematic constraint tree processing
   - Can override entity-based parameters when conflicts exist

2. **Entity-Based Parameters** (High Priority)
   - Direct mappings from resolved entities (people, genres, companies)
   - Core query intent representation
   - Foundation for API parameter structure

3. **Semantic Inference Parameters** (Medium Priority)
   - Contextually inferred from query language patterns
   - Rating preferences, year implications, language hints
   - Only applied when not already specified

4. **Default Fallback Parameters** (Lowest Priority)
   - System defaults for API completeness
   - Applied only when no other source provides values
   - Ensures valid API calls in edge cases

## Conflict Resolution Strategies

### Date Parameter Conflicts
**Problem**: Single year vs. date range parameters  
**Solution**: Date ranges take precedence, single year parameters removed
```python
if has_date_range:
    step["parameters"].pop('primary_release_year', None)
```

### Revenue Constraint Integration  
**Problem**: Revenue entities need specialized parameter mapping  
**Solution**: Revenue-aware constraint building with operator detection
```python
if ent_type == "revenue":
    param_key, priority = self._map_revenue_constraint(ent)
    # Specialized handling for revenue thresholds + strategic sorting
```

### Semantic vs. Explicit Parameters
**Problem**: Inferred parameters conflicting with explicit constraints  
**Solution**: Semantic parameters only fill gaps, never override explicit values
```python
if param_name not in step["parameters"]:  # Only add if missing
    step["parameters"][param_name] = inferred_value
```

## Implementation Architecture

### Core Parameter Injection Flow
```python
def plan(state: AppState) -> AppState:
    # 1. Base execution steps from semantic endpoint matching
    execution_steps = convert_matches_to_execution_steps(...)
    
    # 2. Constraint-based parameter injection (Phase 2)
    for step in execution_steps:
        for constraint in state.constraint_tree.flatten():
            step["parameters"][constraint.key] = str(constraint.value)
    
    # 3. Semantic parameter inference (Phase 3)  
    optional_params = PlanValidator().infer_semantic_parameters(state.input)
    for step in execution_steps:
        for param_name in optional_params:
            if param_name not in step["parameters"]:
                step["parameters"][param_name] = inferred_value
```

### Revenue Constraint Specialization
```python
def _map_revenue_constraint(self, ent: dict) -> tuple:
    """Specialized parameter mapping for revenue constraints"""
    operator = ent.get("operator", "less_than") 
    
    if operator in ("less_than", "less_than_equal"):
        ent["sort_value"] = "popularity.desc"  # Strategic sorting
        ent["threshold"] = int(ent.get("name", 0))  # Post-filter threshold
        ent["threshold_operator"] = operator
        
    return "sort_by", priority=1  # High priority for revenue constraints
```

## Consequences  

### Positive
- **Systematic Parameter Building**: Clear, predictable parameter injection across query complexity
- **Conflict Resolution**: Well-defined precedence rules prevent parameter ambiguity  
- **Extensible Architecture**: New constraint types easily integrate into existing phases
- **Validation Integration**: Parameter conflicts resolved systematically before API calls
- **Performance Optimization**: Strategic parameter selection improves API response relevance

### Negative
- **Pipeline Complexity**: Four-phase system requires careful coordination and testing
- **Debugging Difficulty**: Parameter source tracking across multiple phases can be complex
- **Parameter Interdependency**: Changes in one phase can affect downstream parameter selection

### Mitigations
- **Comprehensive Logging**: Track parameter sources through each injection phase
- **Unit Test Coverage**: Validate parameter injection for each constraint type and combination
- **Parameter Source Attribution**: Maintain metadata about parameter origins for debugging
- **Integration Testing**: End-to-end validation of parameter injection → API calls → results

## Implementation Files

- `app.py` - Main parameter injection orchestration across all phases
- `core/embeddings/hybrid_retrieval.py` - Phase 1: Entity-based parameter mapping
- `core/model/constraint.py` - Phase 2: Constraint-based parameter building with specialized revenue logic  
- `core/planner/plan_validator.py` - Phase 4: Revenue constraint parameter injection with strategic sorting
- `data/param_to_entity_map.json` - Parameter mapping configuration including revenue → sort_by

## Validation Examples

### Simple Entity Query
Input: "Movies starring Tom Hanks"  
Parameters: `{"with_people": "31"}`  
Source: Phase 1 (Entity-based)

### Complex Constraint Query  
Input: "2010s horror movies with revenue under $25M, highly rated"  
Parameters: 
```json
{
  "with_genres": "27",
  "primary_release_date.gte": "2010-01-01", 
  "primary_release_date.lte": "2019-12-31",
  "sort_by": "popularity.desc",
  "vote_count.gte": "50"
}
```
Sources: Phase 1 (genre), Phase 2 (date range, revenue), Phase 3 (rating inference)