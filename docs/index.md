# RTF AI Research Portfolio

Welcome to the professional hub for **RTF AI research** into LLM-driven developer tooling, symbolic/semantic retrieval, and multi-step planning systems.

This portfolio demonstrates a structured research approach with reproducible demos, architecture decisions, and performance analysis across two comprehensive phases:

## Project Overview

- **Phase 1 – TMDB Chatbox**: Natural language movie/TV query system with multi-entity constraint solving, RAG API endpoint routing using ChromaDB, and progressive constraint relaxation
- **Phase 2 – NetBox MCP Server**: Official NetBoxLabs MCP server with 3 generic tools (get_objects, get_object_by_id, get_changelogs) providing shared infrastructure for agent frameworks
- **Phase 3 – OpenAI Orchestration (FAILED)**: Multi-agent orchestration attempt with 0% success rate - see ADR-0013 for failure analysis
- **Phase 4 – Agent Framework Comparison**: Empirical comparison of Deepagents (LangChain) vs Claude SDK approaches to building production NetBox agents

## Research Highlights

- **Hybrid Retrieval**: Combining semantic similarity with symbolic metadata filters for intelligent tool/endpoint selection
- **Performance Optimization**: Field filtering and generic tool patterns for token efficiency
- **Architecture Pattern**: Extraction → Retrieval → Planning → Execution → Validation → Formatting
- **Phase 4 Completion**: Empirical framework comparison validating context-dependent framework selection
- **Future Roadmap**: Neo4j graph integration (Phase 5), RAG intelligence (Phase 6), and analytics platform (Phase 7)

## Implementation Repositories

- **Phase 1**: [TMDB Chatbox](https://github.com/FinnMacCumail/tmdbGPT) - Natural language movie/TV query system
- **Phase 2**: [NetBoxLabs MCP Server](https://github.com/netboxlabs/netbox-mcp-server) - Official NetBox MCP server infrastructure
- **Phase 4A**: [Deepagents Implementation](https://github.com/FinnMacCumail/deepagents) - LangChain-based flexible framework
- **Phase 4B**: [Claude SDK Implementation](https://github.com/FinnMacCumail/claude-agentic-netbox) - Anthropic SDK production framework

## Research Timeline
```mermaid
timeline
    title RTF AI Research Timeline (2025)
    2025-01 : Project Kickoff : LLM landscape survey : Research questions defined
    2025-01–04 : Phase 1 Development : TMDB API integration : Multi-entity constraint solving : Semantic + symbolic retrieval
    2025-04 : Phase 1 Complete : Progressive constraint relaxation : Role-aware validation system
    2025-05–06 : Phase 2 Research : NetBoxLabs MCP server evaluation : Generic tool pattern analysis
    2025-07–08 : Production Integration : NetBox MCP server deployment : Field filtering optimization : Token efficiency improvements
    2025-08 : Documentation Phase : Professional portfolio creation : GitHub Pages deployment
    2025-09 : Phase 3 Failed : OpenAI multi-agent orchestration : 0% success rate : Documented failure analysis
    2025-10 : Phase 4 Complete : Agent framework comparison study : Deepagents vs Claude SDK : Empirical validation
    2025-11+ : Future Development : Neo4j graph integration : RAG-powered intelligence : Analytics platform
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
