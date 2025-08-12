# ADR-0011 — Temporal Sorting and Constraint Interaction Pattern

## Context

The timeline query fix in tmdbGPT Phase 1 revealed complex interactions between temporal sorting logic and constraint validation systems. Timeline queries like "First movies by Steven Spielberg" required both:

- **Temporal intent detection**: Recognizing "First" as chronological ordering requirement
- **Constraint validation bypass**: Person credit summaries needed special handling
- **Sorting preservation**: Temporal ordering must survive constraint filtering
- **Response format routing**: Timeline queries need timeline renderer, not summary renderer

The investigation demonstrated that temporal queries represent a unique class of user intent that crosses traditional architectural boundaries, requiring coordination between multiple processing layers.

## Decision

Establish **temporal sorting and constraint interaction pattern** that preserves chronological ordering through constraint validation while maintaining semantic correctness.

**Core Pattern**:
- Temporal intent detection at LLM extraction layer
- Sort parameter injection during step planning 
- Constraint validation awareness of temporal context
- Timeline response format preservation throughout pipeline
- Endpoint-aware handling for person credit temporal queries

**Technical Implementation**:

```python
# Layer 1: Intent Detection and Sort Parameter Injection
def analyze_temporal_intent(query):
    temporal_indicators = {
        'first': {'sort_by': 'release_date.asc', 'limit': 20},
        'latest': {'sort_by': 'release_date.desc', 'limit': 20},
        'earliest': {'sort_by': 'release_date.asc', 'limit': 10},
        'recent': {'sort_by': 'release_date.desc', 'limit': 15}
    }
    
    for indicator, params in temporal_indicators.items():
        if indicator in query.lower():
            return params
    return None

# Layer 2: Response Format Coordination  
def extract_entities_and_intents(query):
    result = llm_extraction(query)
    
    # Ensure timeline questions use timeline format
    if result.get("question_type") == "timeline" and result.get("response_format") != "timeline":
        result["response_format"] = "timeline"
    
    return result

# Layer 3: Constraint Validation with Temporal Context
def filter_symbolic_responses(state, summaries, endpoint):
    is_person_credits = '/person/' in endpoint and '/movie_credits' in endpoint
    
    if is_person_credits:
        # Timeline queries need bypass for proper temporal ordering
        constraints = getattr(state.constraint_tree, 'constraints', [])
        is_single_person = (len(person_constraints) == 1 and len(constraints) == 1)
        
        if is_single_person:
            # Preserve temporal sorting through constraint bypass
            return [s for s in summaries if s.get("final_score", 0) > 0]
    
    # Standard constraint validation for non-temporal queries
    return apply_constraint_validation(state, summaries)

# Layer 4: Timeline Rendering with Temporal Ordering
def format_timeline(state):
    entries = []
    for item in state.responses:
        if isinstance(item, dict) and item.get("type") == "movie_summary":
            # Extract temporal sorting key
            year = item.get("release_date", "")[:4] if "release_date" in item else None
            entries.append({
                "title": item.get("title"),
                "overview": item.get("overview"),
                "release_year": int(year) if year and year.isdigit() else None,
                "source": item.get("source"),
                "score": item.get("final_score", 1.0)
            })
    
    # Apply chronological sorting
    entries.sort(key=lambda x: x.get("release_year") or 3000)
    return {"response_format": "timeline", "entries": entries}
```

## Consequences

### Positive
- **Temporal Query Success**: "First movies by Steven Spielberg" returns 332 chronologically sorted entries (Firelight 1964 → recent films)
- **Multi-Layer Coordination**: Temporal intent preserved from extraction through rendering
- **Sorting Integrity**: Chronological ordering maintained through constraint validation bypass
- **Response Format Accuracy**: Timeline queries consistently use timeline renderer
- **Pattern Reusability**: Approach applicable to other temporal query types ("Latest", "Earliest")

### Negative  
- **Cross-Layer Complexity**: Temporal handling spans multiple architectural layers
- **Coordination Overhead**: Requires synchronization between intent detection, constraint validation, and rendering
- **Edge Case Management**: Must handle queries with both temporal and non-temporal constraints

### Performance Impact
- **Sorting Efficiency**: Timeline renderer sorts 332 entries by release year with minimal overhead
- **Constraint Bypass**: Single-person temporal queries avoid expensive constraint validation
- **Memory Usage**: Timeline format preserves essential metadata without bloating response size

## Compliance

This pattern follows established software architecture principles:
- **Separation of Concerns**: Each layer handles its specific aspect of temporal processing
- **Data Flow Integrity**: Temporal context preserved through processing pipeline
- **Semantic Consistency**: System behavior aligns with user temporal intent expectations
- **Performance Optimization**: Constraint bypass reduces unnecessary processing for temporal queries

## Related Decisions

- Builds on ADR-0009 endpoint-aware validation by incorporating temporal context into constraint bypass logic
- Extends ADR-0010 debugging methodology with temporal-specific diagnostic patterns
- Complements existing intent-aware sorting system with constraint interaction awareness
- Establishes foundation for future temporal query enhancements (date ranges, period queries)

## Pattern Validation

**Temporal Query Processing Flow**:
```
"First movies by Steven Spielberg"
  ↓
1. LLM Extraction: question_type=timeline, response_format=timeline
2. Intent Analysis: Temporal detected → sort_by=release_date.asc  
3. Entity Resolution: Steven Spielberg → person_id=488
4. Step Planning: /person/488/movie_credits with release_date.asc
5. API Execution: 396 movies retrieved, temporally sorted
6. Constraint Filtering: Single-person bypass applied → 332 movies preserved
7. Timeline Rendering: Chronological display (1964 Firelight → 2024 films)
  ↓
✅ Result: 332 chronologically ordered entries with proper temporal context
```

**Multi-Constraint Temporal Example**:
```  
"Latest horror movies by James Wan"
  ↓
1. LLM Extraction: question_type=timeline, multiple constraints detected
2. Intent Analysis: Temporal (latest) + Genre (horror) constraints
3. Constraint Validation: Full validation applied (no bypass)
4. Temporal Sorting: release_date.desc with genre filtering
5. Timeline Rendering: Recent horror films only
  ↓
✅ Result: Temporal ordering preserved with proper constraint filtering
```

This pattern successfully resolves the tension between temporal user intent and constraint validation requirements while maintaining architectural coherence across processing layers.