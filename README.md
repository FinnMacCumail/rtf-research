# RTF AI Research Portfolio (2025)

## Research Overview: Systematic Anti-Hallucination in Domain-Specific LLM Applications

This repository documents RTF's comprehensive 6-month research program addressing the critical challenge of **LLM hallucination in domain-specific applications**. Through systematic development of constraint-based validation, multi-protocol tool orchestration, and enterprise safety controls, this research demonstrates how to achieve reliable, factually accurate LLM responses in complex technical domains.

### Core Research Hypothesis
*Multi-stage validation combining symbolic constraints, structured tool protocols, and progressive verification can eliminate fabricated responses while maintaining natural language interaction quality.*

### Research Evolution Overview
The research progressed through two comprehensive phases, each building sophisticated anti-hallucination mechanisms:

- **Phase 1 – TMDB RAG + Constraint-Based Validation System**
  - **Repository**: https://github.com/FinnMacCumail/tmdbGPT
  - **Anti-Hallucination Focus**: RAG semantic search, symbolic constraint trees, progressive relaxation, weighted endpoint selection
  - **Key Innovation**: Multi-stage RAG + API validation pipeline preventing fabricated movie/TV data

- **Phase 2 – NetBox MCP Protocol Integration**  
  - **Repository**: https://github.com/FinnMacCumail/mcp-netbox
  - **Anti-Hallucination Focus**: Enterprise safety controls, dual-tool validation, structured protocol enforcement
  - **Key Innovation**: Claude Code API + MCP integration with 142+ validated tools across DCIM/Virtualization/IPAM

👉 **Full technical documentation:** https://finnmaccumail.github.io/rtf-research/

## Research Methodology: Multi-Protocol Anti-Hallucination System

### Problem Definition
Traditional LLM applications suffer from hallucination—generating plausible but factually incorrect responses, particularly problematic in technical domains where accuracy is critical. This research systematically addresses hallucination through structured validation approaches.

### Phase 1: TMDB RAG + Constraint-Based Validation Pipeline

**Query Processing Architecture:**
```
User Query → Intent Recognition → RAG Semantic Search → Entity Extraction → Constraint Tree → Weighted Endpoint Selection → API Validation → Response
```

**Intent Recognition & Classification:**
- **LLM + SpaCy Integration**: Hybrid approach using language model understanding with structured linguistic analysis
- **Query Type Classification**: Movie search, person lookup, relationship queries, temporal constraints
- **Ambiguity Detection**: Identifies unclear queries requiring user clarification to prevent assumption-based responses

**RAG Semantic Search Process:**
- **ChromaDB Vector Database**: Semantic search across movie/TV titles, actor names, and genre metadata
- **Sentence Transformers**: Text-to-vector conversion enabling similarity matching for entity resolution
- **Embedding-Based Disambiguation**: Vector similarity scores resolve ambiguous entity references (e.g., multiple actors with same name)
- **Semantic Grounding**: Retrieved embeddings provide factual context before constraint building, preventing fabricated entity relationships

**Entity Recognition & Validation:**
- **RAG-Enhanced Named Entity Extraction**: Actors, directors, movies, genres validated through semantic similarity scoring
- **Multi-Stage Disambiguation**: ChromaDB similarity search + contextual validation + TMDB database cross-reference
- **Vector-Verified Entity Resolution**: All entities must pass both semantic similarity threshold (>0.8) and database validation

**Weighted Endpoint Selection Algorithm:**
- **Decision Matrix**: Routes between `/discover/movie`, `/person`, `/search` based on:
  - Query complexity coefficient (0.0-1.0)
  - Entity confidence levels (weighted average)
  - Expected result quality metrics
  - API endpoint performance characteristics
  - Weighted Endpoint Selection: Static scoring system optimizes API
  routing using semantic similarity and rule-based parameter matching

**Progressive Constraint Relaxation:**
- **Constraint Hierarchy**: Primary (must match) → Secondary (should match) → Tertiary (nice to have)
- **Systematic Fallbacks**: Expands date ranges, relaxes genre constraints, broadens search scope
- **Provenance Logging**: Complete audit trail of which constraints were modified and why

### Phase 2: NetBox MCP + Claude Code API Integration

**MCP Protocol Architecture:**
- **Tool Discovery Process**: Claude Code API automatically discovers 142+ NetBox MCP tools
- **Capability Advertisement**: Each tool declares parameters, return types, and safety constraints
- **Dynamic Registration**: Real-time tool availability detection and capability matching

**Claude Code API Integration Patterns:**
- **Query-to-Tool Mapping**: Natural language intent parsed and matched to appropriate MCP tools
- **Parameter Validation Pipeline**: Input sanitization, type checking, constraint validation
- **Multi-Tool Orchestration**: Intelligent sequencing of tool calls for complex queries
- **Response Aggregation**: Structured data combination with conflict resolution

**Read-Only Tool Execution Safety:**
- **Idempotent Operations**: All read tools guaranteed to have no side effects
- **Dual-Tool Validation**: "info" + "list_all" tools provide cross-verification
- **Structured Response Enforcement**: MCP schema validation prevents fabricated data structures
- **Context-Aware Safety Levels**: Environment detection (demo/staging/production) with appropriate controls

**Anti-Hallucination MCP Mechanisms:**
- **Real-Time Data Binding**: Direct NetBox API integration eliminates data staleness
- **Atomic Validation**: Read-validate-confirm pattern for all infrastructure queries  
- **Enterprise Safety Controls**: Mandatory confirmation for write operations, comprehensive audit logging
- **Automatic Rollback**: Complex operations with built-in failure recovery

### Cross-Phase Validation Results

**Quantitative Anti-Hallucination Metrics:**
- **NetBox Phase**: 100% tool accessibility with structured response validation
- **Performance Optimization**: 127 API calls → 3 calls through intelligent tool orchestration (97.6% reduction)
- **Enterprise Safety**: Zero fabricated infrastructure data across production environments

**Key Research Innovations:**
1. **RAG + Constraint Hybrid Architecture**: Semantic retrieval combined with symbolic validation preventing fabricated responses
2. **Multi-Stage Entity Validation**: ChromaDB semantic search + database cross-reference + constraint verification
3. **Progressive Relaxation with Provenance**: Systematic fallbacks maintaining accuracy audit trails
4. **MCP Protocol Safety Integration**: Structured tool protocols eliminating response fabrication
5. **Multi-Protocol RAG Orchestration**: Seamless integration of semantic retrieval with enterprise tool safety

## Technical Implementation Highlights

- **RAG + Constraint Hybrid Architecture**: ChromaDB vector database with sentence transformers providing semantic grounding before API validation
- **Multi-Stage Entity Validation**: Semantic similarity scoring (>0.8 threshold) + database cross-reference + constraint verification
- **Vector-Enhanced Endpoint Selection**: Semantic similarity informing intelligent API routing decisions
- **MCP Protocol Integration**: Claude Code API orchestrating 142+ validated tools with enterprise safety controls
- **RAG-Informed Constraint Relaxation**: Semantic search results guide systematic fallback strategies
- **Enterprise Safety Architecture**: Dual-tool validation, atomic operations, and comprehensive audit logging

## Implementation Repositories
- **Phase 1**: TMDB Chatbox – https://github.com/FinnMacCumail/tmdbGPT
- **Phase 2**: NetBox MCP Server – https://github.com/FinnMacCumail/mcp-netbox

## Future Development
A comprehensive four-phase OpenAI orchestration plan exists to transform Phase 2 into an intelligent infrastructure advisor:
1. **Foundation**: GPT-4o-mini intent parsing with LangGraph tool orchestration
2. **Graph Integration**: Neo4j database for instant complex queries (1-3s vs 30s-3min)
3. **RAG Intelligence**: Institutional memory with operational documentation
4. **Analytics Platform**: Predictive capabilities using graph algorithms

**Performance Targets**: 99% cost reduction, 15-50x query performance improvements

