# ADR-0007 — Dual-Mode Financial Query Processing

## Context

Financial queries in the TMDB system present two distinct user intent patterns with different processing requirements:

1. **Constraint-Based Queries**: "Horror movies with revenue under $25 million" - Filtering collections by financial thresholds
2. **Fact-Based Queries**: "What's the revenue for The Blair Witch Project?" - Retrieving specific financial data points

Each pattern requires fundamentally different processing approaches due to:
- **API Endpoint Usage**: Discovery vs. specific movie detail endpoints  
- **Data Extraction**: Collection filtering vs. individual entity fact retrieval
- **Result Formatting**: Ranked lists vs. summary fact responses
- **Performance Characteristics**: Batch processing vs. single entity lookup

## Decision

Implement **dual-mode financial query processing** with separate pipelines optimized for each query type:

### Mode 1: Constraint-Based Financial Filtering
**Query Pattern**: "Movies with revenue [operator] [threshold]"  
**Processing Pipeline**:
```
Entity Extraction → Constraint Building → Strategic Sorting → 
Discovery API → Revenue Data Fetching → Threshold Filtering → Ranked Results
```

**Technical Implementation**:
- Route through constraint system with `question_type: 'list'`
- Build revenue constraints with operator-aware parameter mapping
- Use `/discover/movie` with strategic `sort_by` parameters
- Fetch individual movie details for complete financial data
- Apply post-discovery threshold filtering

### Mode 2: Fact-Based Financial Retrieval  
**Query Pattern**: "What's the revenue for [Movie Title]?"  
**Processing Pipeline**:
```
Entity Extraction → Movie Resolution → Direct Movie API → 
Revenue Extraction → Fact Formatting → Summary Response
```

**Technical Implementation**:
- Route through fact system with `question_type: 'fact'` 
- Bypass constraint system entirely
- Direct `/movie/{id}` API call after entity resolution
- Extract revenue field in `_extract_movie_details()`
- Format using revenue-specific fact templates

**Detection Logic**:
```python
# Revenue question detection in fact formatter
is_revenue_question = any(keyword in query_text for keyword in [
    "revenue", "earnings", "gross", "box office", "made money", "earned"
])

# Revenue constraint detection in entity extractor  
{
    "name": "25000000", 
    "type": "revenue", 
    "operator": "less_than"
}
```

## Rationale

**Why separate processing modes?**
- **Performance Optimization**: Fact queries don't need expensive discovery + filtering pipeline
- **Result Accuracy**: Different data requirements (collections vs. individual facts)
- **User Experience**: Faster response times for simple fact queries
- **API Efficiency**: Minimizes unnecessary API calls based on query intent

**Why fact queries bypass constraints?**
- Fact queries target specific entities, not collections requiring filtering
- Constraint system adds unnecessary complexity for single-entity lookups
- Direct API calls provide faster response times for known entities

**Why constraint queries require post-filtering?**
- TMDB `/discover/movie` doesn't provide revenue filtering parameters
- Revenue data not available in discovery responses
- Threshold filtering requires complete financial data

## Implementation Details

### Constraint-Based Mode Components
```python
# core/model/constraint.py
def _map_revenue_constraint(self, ent: dict) -> tuple:
    # Maps revenue entities to sort_by parameters with thresholds
    
# core/execution/discovery_handler.py  
def handle_discovery_step(step, json_data, state):
    # Fetches revenue data before applying financial filters
    
# core/execution/financial_filter.py
def apply_financial_filters(movies, constraint_tree):
    # Applies threshold-based filtering on complete data
```

### Fact-Based Mode Components
```python
# nlp/nlp_retriever.py
def _extract_movie_details(json_data, endpoint):
    revenue = json_data.get("revenue")
    return [{"revenue": revenue, ...}]
    
# core/formatting/templates.py
def format_fact_response(results, query_text):
    if is_revenue_question and revenue > 0:
        entries.append(f"{title} earned ${revenue:,} in revenue.")
```

### Route Determination
**Constraint Route Triggers**:
- Query contains financial operators: "under", "over", "above", "below"
- Multiple entities present (genre + revenue threshold)
- Collection-oriented language: "movies with revenue..."

**Fact Route Triggers**:  
- Direct financial questions: "what's the revenue of...", "how much did ... earn"
- Single entity focus with no comparative operators
- `question_type: 'fact'` detected by LLM extractor

## Consequences

### Positive
- **Query-Optimized Performance**: Each mode optimized for its specific use case
- **Accurate Results**: Appropriate processing pipeline for different user intents
- **Maintainable Architecture**: Clear separation of concerns between modes
- **Extensible Framework**: Pattern supports future financial query types (budget, profit, etc.)

### Negative
- **Code Complexity**: Dual pipelines require maintenance of separate code paths
- **Testing Overhead**: Both modes require comprehensive test coverage
- **Route Accuracy Dependency**: Incorrect routing degrades user experience

### Mitigations
- **Comprehensive Route Testing**: Validate routing accuracy across query patterns
- **Fallback Mechanisms**: Graceful degradation if primary mode fails
- **Shared Utility Functions**: Common revenue processing logic between modes
- **Documentation**: Clear examples of each mode's appropriate use cases

## Validation Examples

### Constraint-Based Queries (Mode 1)
✅ "Horror movies with revenue under $25 million"  
✅ "Action films that made over $100M"  
✅ "Low budget comedies under $5M revenue"  

### Fact-Based Queries (Mode 2)  
✅ "What's the revenue for The Blair Witch Project?"  
✅ "How much money did Avatar earn?"  
✅ "Give me the box office numbers for Titanic"

## Implementation Files

- `core/llm/extractor.py` - Revenue entity and fact query detection
- `core/formatting/templates.py` - Revenue fact formatting with question detection  
- `nlp/nlp_retriever.py` - Revenue field extraction for fact responses
- `core/execution/discovery_handler.py` - Constraint-based revenue filtering integration
- `app.py` - Dual-mode routing logic in planning phase