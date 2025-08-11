# ADR-0006 — Financial Constraint Parameter Mapping Strategy

## Context

Financial constraint queries like "Horror movies with revenue under $25 million" require mapping revenue thresholds to TMDB API parameters for effective discovery. The challenge lies in the mismatch between user intent and API capabilities:

- **User Intent**: Find movies by revenue thresholds (e.g., "under $25M", "over $100M") 
- **TMDB API Limitation**: `/discover/movie` doesn't support direct revenue filtering parameters
- **API Sorting Options**: Only `sort_by=revenue.asc/desc` available for revenue-based ordering
- **Data Availability**: Revenue data requires individual `/movie/{id}` calls, not available in discovery results

## Decision

Implement a **dual-phase financial constraint mapping strategy** that combines strategic sorting with post-API filtering:

**Phase 1: Strategic Parameter Mapping**
- Map revenue entities to `sort_by` parameters instead of direct filtering
- Use operator-aware sorting strategies:
  - `less_than/less_than_equal` → `sort_by=popularity.desc` (avoids $0 revenue indie films)
  - `greater_than/greater_than_equal` → `sort_by=revenue.desc` (gets highest earners first)

**Phase 2: Post-Discovery Filtering** 
- Fetch individual movie details for revenue data completion
- Apply threshold-based filtering on complete financial data
- Return results matching the original constraint criteria

**Technical Implementation**:
```python
def _map_revenue_constraint(self, ent: dict) -> tuple:
    operator = ent.get("operator", "less_than")
    
    if operator in ("less_than", "less_than_equal"):
        # Strategy: Get popular movies first, then filter by threshold
        ent["sort_value"] = "popularity.desc"
        ent["threshold"] = int(ent.get("name", 0))
        ent["threshold_operator"] = operator
    elif operator in ("greater_than", "greater_than_equal"):
        # Strategy: Get highest revenue movies first
        ent["sort_value"] = "revenue.desc" 
        ent["threshold"] = int(ent.get("name", 0))
        ent["threshold_operator"] = operator
    
    return "sort_by", priority=1
```

## Rationale

**Why not direct revenue parameter filtering?**
- TMDB API doesn't provide revenue filtering parameters in `/discover/movie`
- Direct filtering would require custom API endpoint development

**Why popularity.desc for "under" queries?**
- `sort_by=revenue.asc` returns thousands of $0 revenue independent films first
- Popular movies are more likely to have accurate revenue data
- Provides better user experience with recognizable mainstream titles

**Why individual movie detail fetching?**
- `/discover/movie` responses don't include revenue fields
- Required for accurate threshold-based filtering
- Acceptable performance trade-off for financial query accuracy

## Consequences

### Positive
- **Accurate Financial Filtering**: Users get precise revenue-based results matching their thresholds
- **Strategic API Usage**: Optimizes TMDB API calls by getting relevant movies first
- **Extensible Pattern**: Framework supports future financial constraints (budget, profit margins)
- **Fallback Compatibility**: System gracefully handles missing revenue data

### Negative  
- **Increased API Calls**: Requires individual movie detail fetching (N+1 pattern)
- **Response Latency**: Additional network calls for revenue data completion
- **API Rate Limits**: Higher chance of hitting TMDB rate limits for large result sets

### Mitigations
- **Result Limiting**: Intelligent estimation of required result counts based on constraint selectivity
- **Caching Strategy**: Cache movie detail responses for repeated revenue queries  
- **Progressive Loading**: Fetch revenue data in batches with user-visible progress
- **Fallback Degradation**: Show partial results if API limits are reached

## Implementation Files

- `data/param_to_entity_map.json` - Added `"sort_by": "revenue"` mapping
- `core/model/constraint.py` - Revenue constraint builder with operator logic
- `core/planner/plan_validator.py` - Revenue parameter injection with strategic sorting
- `core/execution/discovery_handler.py` - Individual movie detail fetching before filtering
- `core/execution/financial_filter.py` - Threshold-based revenue filtering implementation