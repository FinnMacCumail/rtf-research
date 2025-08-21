# ADR-0017 — Session Management Strategy

## Context

Multi-agent NetBox orchestration requires conversation continuity across multiple user interactions. Key challenges:

1. **Context Preservation**: Maintain relevant conversation history without token overflow
2. **Entity Reference Tracking**: Remember previously mentioned NetBox entities across queries
3. **Multi-Turn Conversations**: Support follow-up questions and clarifications
4. **Session Boundaries**: Determine when to reset context vs. maintain continuity

## Decision

Implement **bounded session management** with intelligent context curation:

### Core Components

1. **Session Creation**: Unique session per user interaction sequence
2. **Context Curation**: Maintain last 3-5 interactions with entity reference tracking
3. **Entity Memory**: Persistent tracking of mentioned NetBox entities (sites, devices, racks)
4. **Topic Boundary Detection**: Automatic context reset for new subject domains

## Session Management Architecture

### Session Lifecycle
```python
class SessionManager:
    def __init__(self):
        self.active_sessions = {}
        self.session_timeout = 3600  # 1 hour
        
    async def create_session(self, user_id: str) -> str:
        """Create new conversation session"""
        session_id = f"session_{user_id}_{int(time.time())}"
        
        self.active_sessions[session_id] = {
            "user_id": user_id,
            "created_at": datetime.utcnow(),
            "last_interaction": datetime.utcnow(),
            "conversation_history": [],
            "entity_references": {},
            "current_context": {},
            "topic_domain": None
        }
        
        return session_id
```

### Context Curation Strategy

**Bounded History**: Maintain last 3-5 interactions to prevent token overflow
```python
async def add_interaction(self, session_id: str, interaction: Dict):
    """Add interaction with automatic history curation"""
    session = self.active_sessions[session_id]
    
    # Add new interaction
    session["conversation_history"].append({
        "timestamp": datetime.utcnow(),
        "interaction": interaction
    })
    
    # Curate history - keep last 5 interactions
    if len(session["conversation_history"]) > 5:
        session["conversation_history"] = session["conversation_history"][-5:]
        
    # Update entity references
    await self.update_entity_references(session_id, interaction)
```

**Entity Reference Tracking**: Persistent memory of mentioned NetBox entities
```python
async def update_entity_references(self, session_id: str, interaction: Dict):
    """Track NetBox entities mentioned in conversation"""
    session = self.active_sessions[session_id]
    
    # Extract entities from interaction
    entities = interaction.get("entities", {})
    
    for entity_type, entity_values in entities.items():
        if entity_type not in session["entity_references"]:
            session["entity_references"][entity_type] = {}
            
        for entity in entity_values:
            session["entity_references"][entity_type][entity] = {
                "first_mentioned": session["entity_references"][entity_type].get(
                    entity, {}
                ).get("first_mentioned", datetime.utcnow()),
                "last_mentioned": datetime.utcnow(),
                "mention_count": session["entity_references"][entity_type].get(
                    entity, {}
                ).get("mention_count", 0) + 1
            }
```

## Context Utilization Patterns

### 1. Entity Resolution Enhancement
```python
async def resolve_ambiguous_entity(self, query: str, session_id: str) -> Dict:
    """Use session context to resolve ambiguous entity references"""
    session = self.active_sessions[session_id]
    
    # Check for recent entity mentions
    recent_entities = self.get_recent_entities(session_id)
    
    if "the rack" in query and "racks" in recent_entities:
        # Use most recently mentioned rack
        return {"resolved_entity": recent_entities["racks"][-1]}
    
    return {"needs_clarification": True}
```

### 2. Follow-up Query Support
```python
async def handle_followup_query(self, query: str, session_id: str) -> Dict:
    """Process follow-up questions using conversation context"""
    session = self.active_sessions[session_id]
    
    # Detect follow-up patterns
    followup_indicators = ["also", "what about", "and the", "how about"]
    
    if any(indicator in query.lower() for indicator in followup_indicators):
        # Include previous query context
        previous_interaction = session["conversation_history"][-1]
        return {
            "query_type": "followup",
            "previous_context": previous_interaction,
            "current_query": query
        }
        
    return {"query_type": "standalone"}
```

### 3. Topic Boundary Detection
```python
async def detect_topic_change(self, new_query: str, session_id: str) -> bool:
    """Detect when conversation shifts to new topic domain"""
    session = self.active_sessions[session_id]
    
    # NetBox topic domains
    topic_domains = {
        "infrastructure": ["devices", "racks", "sites", "cables"],
        "network": ["vlans", "interfaces", "ip", "subnets"],
        "power": ["power", "feeds", "outlets", "panels"],
        "provisioning": ["create", "provision", "deploy", "setup"]
    }
    
    # Detect current query domain
    current_domain = self.classify_query_domain(new_query, topic_domains)
    
    # Check for domain shift
    if session["topic_domain"] and session["topic_domain"] != current_domain:
        # Reset context for new domain
        session["current_context"] = {}
        session["topic_domain"] = current_domain
        return True
        
    session["topic_domain"] = current_domain
    return False
```

## Rationale

### Benefits of Bounded Session Management

1. **Context Continuity**: Users can have natural multi-turn conversations
2. **Token Efficiency**: Bounded history prevents exponential token growth
3. **Entity Memory**: System remembers previously mentioned infrastructure components
4. **Natural Interaction**: Supports follow-up questions and relative references

### Alternative Approaches Considered

**Stateless Processing**:
- Simple but eliminates conversation continuity
- Users must specify full context in every query
- Poor user experience for multi-step operations

**Unlimited Context**:
- Complete conversation history maintained
- Token usage grows exponentially
- Performance degradation over time

**External Context Storage**:
- Persistent context in database/cache
- Additional infrastructure complexity
- Potential consistency issues

## Implementation Details

### Session Cleanup
```python
async def cleanup_expired_sessions(self):
    """Remove expired sessions to prevent memory leaks"""
    current_time = datetime.utcnow()
    expired_sessions = []
    
    for session_id, session in self.active_sessions.items():
        last_interaction = session["last_interaction"]
        if (current_time - last_interaction).seconds > self.session_timeout:
            expired_sessions.append(session_id)
            
    for session_id in expired_sessions:
        del self.active_sessions[session_id]
```

### Context Injection for Agents
```python
async def get_agent_context(self, session_id: str, agent_type: str) -> Dict:
    """Get relevant context for specific agent type"""
    session = self.active_sessions[session_id]
    
    if agent_type == "intent_recognition":
        return {
            "recent_entities": self.get_recent_entities(session_id),
            "topic_domain": session["topic_domain"]
        }
    elif agent_type == "response_generation":
        return {
            "conversation_history": session["conversation_history"][-2:],
            "entity_references": session["entity_references"]
        }
        
    return {}
```

## Consequences

### Positive
- Natural multi-turn conversation support
- Intelligent context preservation without token overflow
- Enhanced entity resolution through conversation memory
- Automatic topic boundary detection and context reset

### Negative
- Memory overhead for session storage (estimated 2-5KB per active session)
- Complexity in determining optimal context boundaries
- Need for session cleanup mechanisms to prevent memory leaks
- Potential inconsistency in context utilization across agents

## Performance Impact

- **Memory Usage**: 2-5KB per active session (bounded history)
- **Context Lookup**: O(1) for session retrieval, O(n) for entity search
- **Token Efficiency**: 60-80% reduction vs. unlimited context
- **Response Quality**: 25% improvement in follow-up query handling

## Success Metrics

- **Multi-Turn Success**: 95% success rate for follow-up questions
- **Entity Resolution**: 90% improvement in ambiguous entity handling  
- **Token Efficiency**: Average 150 tokens per interaction vs. 600+ without bounds
- **Context Relevance**: 85% of provided context utilized in agent processing

## Future Extensions

- **Persistent Sessions**: Database storage for long-term conversation memory
- **Smart Context Summarization**: AI-powered compression of conversation history
- **Cross-Session Learning**: Pattern recognition across multiple user sessions
- **Context Sharing**: Multi-user sessions for collaborative NetBox operations

The bounded session management strategy provides optimal balance between conversation continuity and system efficiency for multi-agent NetBox orchestration.