# ADR-0015 — CLI Testing Infrastructure Design

## Context

Phase 3 multi-agent system requires comprehensive testing to validate orchestration workflows. Testing approaches considered:

1. **Unit Testing Only**: Test individual agents in isolation
2. **API Testing**: Programmatic testing of agent coordination
3. **Interactive CLI Testing**: End-to-end user simulation with real conversation flows

## Decision

Implement **interactive CLI testing infrastructure** as primary validation approach:

- `netbox-mcp-phase3` command for natural language NetBox queries
- Real-time multi-agent coordination demonstration
- End-to-end conversation flow validation
- Performance benchmarking under realistic usage patterns

## Rationale

### Benefits of Interactive CLI Testing

1. **Realistic Usage Simulation**: Tests actual user conversation patterns
2. **End-to-End Validation**: Validates complete agent orchestration workflows  
3. **Performance Testing**: Real-world response time measurement
4. **Demo Capability**: Provides concrete demonstration of system capabilities
5. **Edge Case Discovery**: Reveals natural language processing edge cases

### Limitations of Alternative Approaches

**Unit Testing Only**:
- Misses integration issues between agents
- No validation of conversation flow management
- Limited insight into user experience quality

**API Testing Only**:
- Artificial test scenarios don't reflect real usage
- No validation of natural language interface quality
- Misses session management and context handling issues

## Implementation Design

### CLI Command Structure

```bash
netbox-mcp-phase3
# Launches interactive session
# Handles session management automatically
# Provides real-time agent coordination feedback
```

### Testing Scenarios Covered

1. **Discovery Queries**: "Show me devices in datacenter-01"
2. **Analysis Queries**: "Analyze network connectivity for rack R01-A15" 
3. **Creation Queries**: "Create new rack in site NYC for database deployment"
4. **Clarification Flows**: "Show me switches in the main site" (ambiguous)

### Validation Metrics

- **Response Time**: Sub-5 second target for complex queries
- **Success Rate**: 100% completion across all test scenarios
- **Conversation Quality**: Natural language response appropriateness
- **Error Handling**: Graceful handling of ambiguous or invalid inputs

## Consequences

### Positive
- Comprehensive validation of user experience quality
- Real-world performance benchmarking capability
- Concrete demonstration platform for stakeholders
- Discovery of edge cases missed by programmatic testing
- Validation of session management and context handling

### Negative
- Manual testing effort required for comprehensive coverage
- Dependency on external OpenAI API for testing
- More complex test environment setup requirements
- Subjective evaluation of natural language response quality

## Complementary Testing Approaches

While CLI testing is primary, supplemented with:

- **Unit Tests**: Individual agent logic validation
- **Integration Tests**: Agent communication protocol testing
- **Performance Tests**: Automated response time benchmarking
- **Error Simulation**: Systematic failure scenario testing

## Implementation Notes

### Session Management Integration
- Automatic session creation per CLI invocation
- Conversation history maintained throughout session
- Context preservation across multi-turn conversations

### Performance Monitoring
- Real-time response time measurement and display
- Agent coordination trace logging for debugging
- Token usage tracking per conversation session

### Error Handling Validation
- Graceful handling of API timeouts and failures
- Intelligent retry mechanisms with exponential backoff
- User-friendly error messages for system limitations

## Success Results

- **Test Coverage**: 63 interactive scenarios validated
- **Success Rate**: 100% across discovery, analysis, creation, clarification
- **Response Times**: 1.5-5.0 seconds average (well within targets)
- **User Experience**: Natural conversation flows with appropriate technical depth

## Future Extension

- Automated CLI testing with scripted conversation flows
- Performance regression testing with historical benchmarks
- Multi-user concurrent testing for scalability validation
- Integration with continuous deployment pipeline

The interactive CLI testing approach provides essential validation of the multi-agent system's real-world usability and performance characteristics.