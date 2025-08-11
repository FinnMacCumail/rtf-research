# ADR-0005 — Entity Resolution Cache Override Pattern

## Context

Entity resolution in entertainment domains presents unique disambiguation challenges:
- **Geographic Ambiguity**: "BBC" could refer to UK BBC One (ID: 4) or BBC Japan (ID: 4796)
- **Brand Localization**: "Hulu" has regional variants with different content catalogs
- **User Expectation**: Users typically expect primary/origin entities for well-known brands
- **Cache Conflicts**: Generic entity caching can resolve to less relevant regional variants

**Specific Problem**:
- Query "BBC shows" resolved to Japanese BBC, returning anime content instead of UK BBC content
- Query "Hulu originals" sometimes resolved to regional Hulu variants
- Standard entity resolution prioritized first-found matches rather than relevance/origin

## Decision

Implement a **cache override pattern** that allows specific entity disambiguation while preserving general caching performance.

**Technical Implementation**:
```python
# Special BBC handling - override cache for generic 'bbc' 
if name_normalized == "bbc":
    # Update cache to prefer UK BBC One over Japanese BBC
    self.network_cache[name_normalized] = 4  # UK BBC One ID
    return 4

# Similar pattern for other ambiguous entities
if name_normalized == "hulu":
    # Prefer US Hulu over regional variants
    self.network_cache[name_normalized] = 453  # US Hulu ID
    return 453
```

**Alternative Approaches Considered**:
1. **Context-Based Resolution**: Use query context to determine geographic preference
2. **User Preference System**: Allow users to set regional preferences
3. **Popularity-Based Ranking**: Resolve based on content volume or user engagement metrics
4. **Complete Cache Bypass**: Skip caching for ambiguous entities

**Decision Rationale**:
- **Performance Preservation**: Maintains caching benefits for unambiguous entities
- **Predictable Behavior**: Consistent resolution for common ambiguous cases
- **Minimal Complexity**: Simple override pattern vs. complex disambiguation logic
- **Domain Knowledge**: Leverages entertainment industry knowledge about brand origins

## Consequences

### Positive
- **Improved User Experience**: Queries return expected content from primary brand entities
- **Predictable Entity Resolution**: Consistent disambiguation for well-known brands
- **Performance Efficiency**: Preserves caching performance for majority of entities
- **Extensible Pattern**: Easily expandable for future ambiguous entities
- **Geographic Relevance**: Better alignment with user expectations about brand origins

### Negative
- **Hard-Coded Preferences**: Regional preferences are embedded rather than configurable
- **Maintenance Overhead**: New ambiguous entities require manual identification and override setup
- **Limited Flexibility**: Cannot easily adapt to different user geographic contexts
- **Knowledge Dependency**: Requires domain knowledge about entity relationships and importance

### Implementation Considerations
- **Entity ID Stability**: Relies on TMDB API entity IDs remaining stable over time
- **Cache Invalidation**: Override decisions are permanent for session duration
- **Scalability**: Pattern suitable for small number of highly ambiguous entities

## Compliance

This approach aligns with software engineering best practices:
- **Principle of Least Surprise**: Users get expected results for common brand queries
- **Performance Optimization**: Selective override preserves general caching benefits
- **Domain-Driven Design**: Solution incorporates entertainment industry domain knowledge

## Future Considerations

**Potential Enhancements**:
- Configuration-driven override rules instead of hard-coded preferences
- Geographic context detection for dynamic regional preferences  
- Popularity metrics integration for data-driven disambiguation
- User preference learning and adaptation over time

**Monitoring Requirements**:
- Track override usage patterns to identify additional disambiguation needs
- Monitor user satisfaction with entity resolution results
- Evaluate impact on cache hit rates and performance metrics

## Related Decisions

- Builds on ADR-0002 chatbox architecture by enhancing the entity resolution component
- Complements ADR-0003 diagnostic methodology by providing specific solution pattern
- Works alongside ADR-0004 media type detection to improve overall query accuracy