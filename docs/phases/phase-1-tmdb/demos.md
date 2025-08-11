# Phase 1 Demo Scenarios

## Reproducible Query Examples

### Multi-Entity Constraint Solving

**Query**: "Movies starring Leonardo DiCaprio directed by Martin Scorsese"

**Processing Flow**:
1. **Entity Extraction**: Identifies `Leonardo DiCaprio` (actor), `Martin Scorsese` (director)
2. **Constraint Building**: Creates AND constraint across role relationships
3. **API Orchestration**: Uses `/discover/movie` with `with_cast` and `with_crew` parameters
4. **Results**: Returns *The Departed*, *Gangs of New York*, *The Wolf of Wall Street*, etc.

**Demonstration Value**: Shows successful multi-role constraint resolution

---

**Query**: "Who created Breaking Bad?"

**Processing Flow**:
1. **Intent Classification**: Identifies as TV show creator query
2. **Entity Resolution**: Resolves "Breaking Bad" to TMDB TV series ID
3. **API Selection**: Routes to `/tv/{id}/credits` endpoint
4. **Role Filtering**: Extracts creator role from crew data
5. **Results**: Returns "Vince Gilligan"

**Demonstration Value**: Demonstrates precise role-aware information extraction

---

**Query**: "Leonardo DiCaprio movies from 2010-2015"

**Processing Flow**:
1. **Entity + Temporal Extraction**: Actor name + date range
2. **Constraint Resolution**: Maps to `with_cast` + `primary_release_date.gte/lte`
3. **Progressive Relaxation**: If no exact matches, expands date range incrementally
4. **Results**: Returns *Inception*, *Django Unchained*, *The Revenant*, etc.

**Demonstration Value**: Shows temporal constraint handling and fallback mechanisms

## Complex Query Demonstrations

### Constraint Relaxation Example

**Initial Query**: "Sci-fi movies with Tom Hanks from 1995"
- **Strict Search**: No direct matches
- **Relaxation Step 1**: Expand to 1990-2000 → Still no matches
- **Relaxation Step 2**: Relax genre to include "thriller" → Finds related films
- **Provenance Log**: Documents which constraints were modified and why

### Multi-Step Planning

**Query**: "Movies starring Matt Damon written by Ben Affleck"

**Execution Plan**:
1. **Entity Extraction**: Identify "Matt Damon" (actor) and "Ben Affleck" (writer)
2. **Multi-Role Constraint Building**: Create AND constraint across actor and writer roles
3. **Context Retrieval**: Gather relevant filmography data for both people
4. **Search Strategy Planning**: Determine optimal API endpoints for multi-role intersection
5. **API Orchestration**: Execute constraint-based discovery with role parameters
6. **Result Processing**: Filter and validate movies matching both constraints
7. **List Formatting**: Present results in numbered list format

**Result Format**:
```
1. Good Will Hunting
2. The Last Duel
3. Air
4. The Instigators
5. Dogma
6. Jersey Girl
7. Field of Dreams
8. Jay and Silent Bob Strike Back
9. Chasing Amy
10. School Ties
11. Netflix Tudum 2025
12. Jay and Silent Bob Reboot
13. The Third Wheel
14. The Roast of Tom Brady
15. Gone Baby Gone
```

**Demonstration Value**: Shows multi-role constraint solving, cross-role relationship discovery, and comprehensive intersection of actor and writer filmographies. Reveals actual Hollywood collaborations and partnerships.

## Setup Instructions

### Prerequisites
```bash
# Python environment setup
python -m venv tmdb-env
source tmdb-env/bin/activate
pip install -r requirements.txt
```

### API Configuration
```bash
# Environment variables required
export TMDB_API_KEY="your_tmdb_api_key"
export OPENAI_API_KEY="your_openai_api_key"
```

### ChromaDB Vector Database Setup

The system requires ChromaDB to be populated with TMDB API endpoint embeddings before first use.

#### 1. Initial Database Population
```bash
# Navigate to project directory
cd /path/to/tmdbGPT

# Populate TMDB API endpoint embeddings (54 endpoints)
python core/embeddings/semantic_embed.py

# Populate parameter search embeddings  
python core/embeddings/embed_tmdb_parameters.py
```

#### 2. Verify ChromaDB Initialization
```bash
# Check that chroma_db directory was created
ls chroma_db/
# Expected: chroma.sqlite3 and collection subdirectories

# Test semantic search functionality
python -c "from core.embeddings.hybrid_retrieval import retrieve_semantic_matches; print(retrieve_semantic_matches('find movies by actor'))"
```

#### 3. Troubleshooting
- **First Run**: Sentence transformer model downloads (~90MB) automatically
- **Permissions**: Ensure write access to project directory for chroma_db creation  
- **Dependencies**: Run `pip install chromadb sentence-transformers` if missing

### Running Demos
```bash
# Execute demo scenarios
python demos/multi_entity_demo.py
python demos/constraint_relaxation_demo.py
python demos/complex_planning_demo.py
```

Each demo includes detailed logging to show the internal decision-making process and constraint resolution steps.