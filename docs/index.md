# RTF AI Research Portfolio

Welcome to the professional hub for **RTF AI research** into LLM-driven developer tooling, symbolic/semantic retrieval, and multi-step planning systems.

This portfolio demonstrates a structured research approach with reproducible demos, architecture decisions, and performance analysis across two comprehensive phases:

## Project Overview

- **Phase 1 – TMDB Chatbox**: Natural language movie/TV query system with multi-entity constraint solving, RAG API endpoint routing using ChromaDB, and progressive constraint relaxation
- **Phase 2 – NetBox MCP Integration**: Enterprise-grade NetBox MCP server with 142+ tools, performance optimization for token overflow and N+1 query issues
- **Phase 3 – OpenAI Agent Orchestration**: **[IN DEVELOPMENT]** Intelligent coordination of existing NetBox MCP tools using OpenAI GPT-4o-mini and LangGraph state machines for 99% cost reduction and graceful limitation handling

## Research Highlights

- **Hybrid Retrieval**: Combining semantic similarity with symbolic metadata filters for intelligent tool/endpoint selection  
- **Performance Optimization**: Solved critical N+1 query patterns (VLAN queries: 127 API calls → optimized batching)
- **Architecture Pattern**: Extraction → Retrieval → Planning → Execution → Validation → Formatting
- **Current Development**: Phase 3 OpenAI orchestration with intelligent tool coordination and graceful limitation handling
- **Future Roadmap**: Neo4j graph integration (Phase 2), RAG intelligence (Phase 3), and analytics platform (Phase 4)

## Implementation Repositories

- **Phase 1**: [TMDB Chatbox](https://github.com/FinnMacCumail/tmdbGPT) - Natural language movie/TV query system
- **Phase 2**: [NetBox MCP Server](https://github.com/FinnMacCumail/mcp-netbox) - Enterprise network operations integration
- **Phase 3**: [NetBox MCP Development](https://github.com/FinnMacCumail/mcp-netbox) - `feature/openai-agent-foundation` branch (IN DEVELOPMENT)

## Research Timeline
```mermaid
timeline
    title RTF AI Research Timeline (2025)
    2025-01 : Project Kickoff : LLM landscape survey : Research questions defined
    2025-01–04 : Phase 1 Development : TMDB API integration : Multi-entity constraint solving : Semantic + symbolic retrieval
    2025-04 : Phase 1 Complete : Progressive constraint relaxation : Role-aware validation system
    2025-05–06 : Phase 2 Research : NetBox MCP evaluation : Performance bottleneck analysis
    2025-07–08 : Production Integration : 142+ MCP tools deployment : N+1 query optimization : Token overflow solutions
    2025-08 : Documentation Phase : Professional portfolio creation : GitHub Pages deployment : Phase 3 strategic planning
    2025-09+ : Phase 3 Development : OpenAI GPT-4o-mini orchestration : LangGraph tool coordination : Graceful limitation handling
    2025-10+ : Future Development : Neo4j graph integration : RAG-powered intelligence : Analytics platform
```

## Core Architecture

The research demonstrates a consistent architectural pattern across both phases:

```mermaid
flowchart TD
    A[User Query] --> B[Intent/Entity Extraction]
    B --> C[Hybrid Retrieval<br/>Semantic + Symbolic]
    C --> D[Planning & Constraint Engine]
    D --> E[Endpoint/Tool Execution]
    E --> F[Validation & Post-processing]
    F --> G[Response Formatting]
    G --> H[Structured Answer]
    
    style A fill:#e1f5fe
    style H fill:#e8f5e8
    style C fill:#fff3e0
    style D fill:#fce4ec
```
