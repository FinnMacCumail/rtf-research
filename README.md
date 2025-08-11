# RTF AI Research Portfolio (2025)

## Research Overview: Systematic Anti-Hallucination in Domain-Specific LLM Applications

This repository documents RTF's comprehensive 6-month research program addressing the critical challenge of **LLM hallucination in domain-specific applications**. Through systematic development of constraint-based validation, multi-protocol tool orchestration, and enterprise safety controls, this research demonstrates how to achieve reliable, factually accurate LLM responses in complex technical domains.

### Core Research Hypothesis
*Multi-stage validation combining symbolic constraints, structured tool protocols, and progressive verification can eliminate fabricated responses while maintaining natural language interaction quality.*

### Research Evolution Overview
The research progressed through two comprehensive phases, each building sophisticated anti-hallucination mechanisms:

- **Phase 1 – TMDB RAG API Routing + Constraint-Based Validation System**
  - **Repository**: https://github.com/FinnMacCumail/tmdbGPT
  - **Anti-Hallucination Focus**: RAG API endpoint routing, symbolic constraint trees, progressive relaxation, TMDB search API validation
  - **Key Innovation**: Multi-stage RAG endpoint routing + API validation pipeline preventing fabricated movie/TV data

- **Phase 2 – NetBox MCP Protocol Integration**  
  - **Repository**: https://github.com/FinnMacCumail/mcp-netbox
  - **Anti-Hallucination Focus**: Enterprise safety controls, dual-tool validation, structured protocol enforcement
  - **Key Innovation**: Claude Code API + MCP integration with 142+ validated tools across DCIM/Virtualization/IPAM

👉 **Full technical documentation:** https://finnmaccumail.github.io/rtf-research/

## Research Methodology: Multi-Protocol Anti-Hallucination System

### Problem Definition
Traditional LLM applications suffer from hallucination—generating plausible but factually incorrect responses, particularly problematic in technical domains where accuracy is critical. This research systematically addresses hallucination through structured validation approaches.

### Phase 1: TMDB RAG API Routing + Constraint-Based Validation Pipeline

**Query Processing Architecture:**
```
User Query → Intent Recognition → RAG API Endpoint Routing → Entity Resolution (TMDB Search API) → Constraint Building → API Execution → Response Validation
```

**Intent Recognition & Classification:**
- **LLM + SpaCy Integration**: Hybrid approach using language model understanding with structured linguistic analysis
- **Query Type Classification**: Movie search, person lookup, relationship queries, temporal constraints
- **Ambiguity Detection**: Identifies unclear queries requiring user clarification to prevent assumption-based responses

**RAG API Endpoint Routing Process:**
- **ChromaDB Vector Database**: Semantic search across 54 TMDB API endpoint descriptions for optimal routing
- **Sentence Transformers**: Query-to-vector conversion using all-MiniLM-L6-v2 model enabling similarity matching between user queries and API capabilities
- **Endpoint-Based Routing**: Vector similarity scores determine which TMDB APIs to call (e.g., /discover vs /search vs /person)
- **Semantic API Matching**: Retrieved endpoint embeddings guide query routing to prevent incorrect API usage

**Entity Resolution & Validation:**
- **TMDB Search API Integration**: Actors, directors, movies, genres resolved through TMDB Search API calls
- **Multi-Stage Entity Resolution**: Search API calls resolve "Tom Hanks" → person_id, then cross-reference validation
- **API-Verified Entity Resolution**: All entities validated through TMDB database lookups with constraint building

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
1. **RAG-Powered API Orchestration**: Novel use of semantic search for API endpoint selection rather than content retrieval
2. **Semantic Query Routing**: Vector similarity matching user queries to optimal TMDB API endpoints from 54 endpoint descriptions
3. **Progressive Relaxation with Provenance**: Systematic fallbacks maintaining accuracy audit trails
4. **MCP Protocol Safety Integration**: Structured tool protocols eliminating response fabrication
5. **Multi-Protocol API Intelligence**: Seamless integration of semantic endpoint routing with enterprise tool safety

## Technical Implementation Highlights

- **RAG API Orchestration**: ChromaDB vector database storing 54 TMDB endpoint descriptions with semantic routing using all-MiniLM-L6-v2
- **Semantic Endpoint Intelligence**: Vector similarity matching queries to API capabilities (e.g., "movies starring actors" → /discover/movie)
- **TMDB API-Based Entity Resolution**: Search API calls resolve entity names to IDs before constraint building
- **MCP Protocol Integration**: Claude Code API orchestrating 142+ validated tools with enterprise safety controls
- **Intelligent Query Routing**: Semantic endpoint selection preventing incorrect API usage patterns
- **Enterprise Safety Architecture**: Dual-tool validation, atomic operations, and comprehensive audit logging

## Implementation Repositories
- **Phase 1**: TMDB Chatbox – https://github.com/FinnMacCumail/tmdbGPT
- **Phase 2**: NetBox MCP Server – https://github.com/FinnMacCumail/mcp-netbox

## Future Development

### Strategic Vision: Intelligent Infrastructure Advisor

A comprehensive four-phase OpenAI orchestration plan exists to transform the NetBox MCP system from a static data repository into an **intelligent infrastructure advisor** through progressive enhancement with modern AI orchestration technologies.

### Architectural Transformation Overview

The transformation progresses through four strategic phases:

1. **Phase 1A: Foundation & Quick Wins**
   - **Objective**: Replace Claude CLI process management with efficient OpenAI GPT-4o-mini intent parsing and LangGraph tool orchestration
   - **Capabilities**: Intelligent query classification, multi-step workflow orchestration, performance optimization
   - **Impact**: Immediate 90% cost reduction + 15x performance improvement

2. **Phase 2: Neo4j Graph Intelligence**
   - **Objective**: Add pre-computed relationship intelligence using Neo4j graph database for instant complex queries
   - **Capabilities**: Graph-based relationship queries, real-time synchronization, hybrid routing intelligence
   - **Impact**: Enable previously impossible relationship queries (20-50x faster complex analysis)

3. **Phase 3: RAG-Powered Semantic Intelligence**
   - **Objective**: Add contextual understanding through operational documentation and institutional memory integration
   - **Capabilities**: Semantic search across documentation, contextual recommendations, operational pattern recognition
   - **Impact**: Transform system into organizational knowledge advisor with historical context

4. **Phase 4: Advanced Analytics Platform**
   - **Objective**: Deploy graph algorithms and predictive analytics for intelligent infrastructure insights
   - **Capabilities**: Network bottleneck identification, capacity planning, predictive maintenance, operational intelligence
   - **Impact**: Predictive capabilities delivering 10-100x operational efficiency gains

### Performance Transformation Targets

- **Cost Optimization**: 99% reduction (from $0.13 to $0.001 per query)
- **Simple Queries**: 15x faster (3-10 seconds → 200ms-1 second)
- **Complex Queries**: 20-50x faster (30 seconds-3 minutes → 1-3 seconds)
- **New Capabilities**: Analytical insights previously impossible with traditional approaches

👉 **Comprehensive Technical Plan**: [Four-Phase Orchestration Details](docs/future-development/orchestration-plan.md)

