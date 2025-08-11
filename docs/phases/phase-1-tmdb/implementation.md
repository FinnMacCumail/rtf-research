# Phase 1 Implementation Details

## Technical Architecture

The TMDB chatbox implements a sophisticated natural language processing pipeline combining semantic understanding with symbolic constraint solving.

### Core Components

**ChromaDB API Endpoint Router**
- Semantic search using sentence transformers (all-MiniLM-L6-v2) for API endpoint selection
- Vector embeddings for 54 TMDB API endpoint descriptions with metadata (media_type, supported_intents, parameters)
- Efficient routing of natural language queries to optimal TMDB API endpoints

**Multi-Entity Constraint Engine**
```python
# Example constraint resolution for complex queries
query = "Movies starring Leonardo DiCaprio directed by Martin Scorsese"
constraints = {
    'actors': ['Leonardo DiCaprio'],
    'directors': ['Martin Scorsese'],
    'media_type': 'movie'
}
# Progressive relaxation if no results found
```

**Progressive Constraint Relaxation**
- Initial strict matching across all detected entities and roles
- Systematic relaxation when no results found (e.g., expand date ranges, relax secondary constraints)
- Provenance logging to track which constraints were modified
- Fallback mechanisms for graceful degradation

### API Integration Patterns

**RAG-Powered Endpoint Selection**
- ChromaDB vector database with 54 TMDB API endpoint descriptions embedded using all-MiniLM-L6-v2
- **Setup Process**: Run `semantic_embed.py` to populate endpoints collection from `data/tmdb.json`
- **Parameter Embeddings**: Run `embed_tmdb_parameters.py` to create parameter search index from `data/tmdb_parameters.json`
- **Query Process**: Natural language → sentence embeddings → vector similarity → ranked endpoint matches
- **Integration**: Combined with parameter-aware reranking and media type boosting for optimal routing
- Example: "movies starring actors" query matches `/discover/movie` endpoint description with highest similarity
- Parameter mapping from TMDB Search API resolved entities to selected endpoint parameters
- Role-aware validation ensuring cast/crew relationships are correctly identified

**Entity Resolution via TMDB Search API**
- Post-routing entity resolution using TMDB Search API calls (e.g., "Tom Hanks" → person_id)
- Constraint building from resolved entity IDs and query parameters
- Template-based formatting (list, timeline, comparison, fact-based responses)

### Performance Optimizations

**Caching Strategy**
- Intelligent caching of TMDB API responses with TTL management
- Endpoint routing cache for similar query patterns using vector similarity
- Entity resolution cache to avoid repeated TMDB Search API person/company lookups

**Batch Processing**
- Parallel API calls for multi-entity queries
- Efficient handling of large result sets with pagination
- Memory-efficient processing of constraint combinations