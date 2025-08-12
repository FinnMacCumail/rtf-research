# ADR-0009 — Endpoint-Aware Constraint Validation

## Context

Timeline queries like "First movies by Steven Spielberg" were systematically failing in tmdbGPT Phase 1, returning "No summary available" despite person credit data being successfully retrieved from TMDB APIs. Investigation revealed a fundamental mismatch between constraint validation logic and person credit endpoint semantics:

- **Constraint validation expectation**: Movies should "contain" the person (via cast/crew metadata)
- **Person credit endpoint reality**: Endpoint returns movies the person was involved in (opposite direction)
- **Dual filtering layers**: Two separate constraint validation functions both filtered out valid person credit results
- **Timeline rendering failure**: Timeline renderer worked correctly but received empty data

This represented a critical architectural gap where the constraint system's assumptions didn't align with certain API endpoint semantics.

## Decision

Implement **endpoint-aware constraint validation** with selective bypass logic for person credit summaries while preserving multi-constraint validation integrity.

**Core Strategy**:
- Detect person credit endpoints via source URL patterns (`/person/{id}/movie_credits`)
- Apply bypass logic only for single-constraint person queries (exactly 1 constraint total)
- Preserve full constraint validation for multi-constraint queries (person + genre, person + person, etc.)
- Implement dual-layer fix covering both filtering points in the pipeline

**Technical Implementation**:
```python
# Discovery Handler Layer
def filter_symbolic_responses(state, summaries, endpoint):
    is_person_credits = '/person/' in endpoint and '/movie_credits' in endpoint
    
    if is_person_credits:
        constraints = getattr(state.constraint_tree, 'constraints', [])
        person_constraints = [c for c in constraints if hasattr(c, 'key') and c.key == 'with_people']
        
        # Only bypass for purely person-based queries
        is_single_person = (len(person_constraints) == 1 and len(constraints) == 1)
        
        if is_single_person:
            # Bypass constraint validation - endpoint already filtered correctly
            return [s for s in summaries if s.get("final_score", 0) > 0]

# Plan Utils Layer  
def filter_valid_movies_or_tv(entities, constraint_tree, registry):
    is_person_credits_data = any(
        '/person/' in str(m.get('source', '')) and '/movie_credits' in str(m.get('source', ''))
        for m in entities[:3] if isinstance(m, dict)
    )
    
    if is_person_credits_data:
        constraints = getattr(constraint_tree, 'constraints', [])
        person_constraints = [c for c in constraints if hasattr(c, 'key') and c.key == 'with_people']
        is_single_person = (len(person_constraints) == 1 and len(constraints) == 1)
        
        if is_single_person:
            # Bypass constraint validation for single-person queries
            return entities
```

## Consequences

### Positive
- **Timeline Query Resolution**: "First movies by Steven Spielberg" returns 332 chronologically sorted entries instead of 0
- **Precision Constraint Handling**: Multi-constraint queries like "Horror movies by James Wan" still apply proper filtering (20 results vs 89 unfiltered)
- **Backward Compatibility**: All existing discovery and list queries remain unaffected
- **Minimal Risk Approach**: Only affects single-person queries from person credit endpoints
- **Architectural Alignment**: Resolves semantic mismatch between constraint expectations and endpoint realities

### Negative
- **Code Complexity**: Introduces endpoint detection logic and dual-layer bypass patterns
- **Maintenance Burden**: Requires coordination between two separate filtering functions
- **Edge Case Handling**: Must carefully distinguish single-person vs multi-constraint scenarios

### Risk Mitigation
- **Strict Detection Logic**: Only bypasses when exactly 1 constraint total (prevents over-broad bypassing)
- **Comprehensive Testing**: Covers both positive cases and regression prevention
- **Clear Documentation**: Explains bypass conditions and architectural reasoning

## Compliance

This decision follows established software engineering patterns:
- **Semantic Correctness**: Aligns system behavior with API endpoint semantics
- **Minimal Intervention**: Implements smallest possible change to fix systematic failure
- **Test-Driven Development**: Uses comprehensive test coverage to prevent regressions
- **Architectural Consistency**: Maintains overall constraint validation architecture while addressing specific endpoint mismatch

## Related Decisions

- Builds on ADR-0003 diagnostic methodology which identified the root cause through systematic execution tracing
- Complements ADR-0004 media type detection by addressing another class of semantic/symbolic processing alignment issues
- Establishes pattern for future endpoint-semantic mismatch resolution scenarios

## Validation Results

**Before Fix**:
```
❌ "First movies by Steven Spielberg" → "No summary available" (0 entries)
❌ Timeline queries systematically failing across all person-based queries
✅ List queries working normally
✅ Discovery queries unaffected
```

**After Fix**:
```
✅ "First movies by Steven Spielberg" → 332 chronologically sorted entries (Firelight 1964 → recent films)
✅ "Horror movies by James Wan" → 20 properly filtered horror films (vs 89 unfiltered)
✅ "Movies starring Leonardo DiCaprio directed by Martin Scorsese" → Multi-constraint validation preserved
✅ All regression tests passing (discovery, company constraints, mixed queries)
✅ Timeline renderer working with proper temporal sorting
```