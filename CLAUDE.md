# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **meta portfolio repository** serving as the professional hub for AI research into LLM-driven developer tooling. It follows a structured approach with:
- Research narrative and timeline documentation
- Architecture Decision Records (ADRs) for technical decisions
- Reproducible demos and evaluation methodologies
- Links to two active phase repositories for implementation details

## Documentation Commands

**Build and serve documentation locally:**
```bash
pip install mkdocs-material
mkdocs serve
```

**Deploy to GitHub Pages:**
```bash
mkdocs gh-deploy --force
```

Documentation is automatically deployed via GitHub Actions on push to `main` branch.

## GitHub Repository Strategy

This repository implements a professional research presentation strategy:
- **Hub Structure**: Central documentation linking to phase-specific implementation repositories
- **GitHub Pages**: Polished documentation site using MkDocs Material theme
- **GitHub Features**: Projects board for roadmap tracking, Releases for milestone summaries, Discussions for research notes
- **Professional Presentation**: Timeline visualization, reproducible instructions, demo scenarios

## Architecture Overview

This repository documents AI research focused on LLM-driven developer tooling across two main phases:

**Core Architecture Pattern (ADR-0002):**
- Extraction → Retrieval → Planning → Execution → Validation → Formatting
- Hybrid retrieval combining semantic similarity with symbolic metadata filters  
- Symbolic planner for endpoint/tool selection based on entities, roles, and constraints
- Constraint trees with AND/OR logic and provenance-aware fallbacks

**Phase 1 - TMDB Chatbox:**
- Natural language movie/TV show query system using TMDB API
- Multi-entity constraint solving with role-aware validation
- ChromaDB RAG API endpoint routing using sentence transformers
- Progressive constraint relaxation with comprehensive logging
- Repository: https://github.com/FinnMacCumail/tmdbGPT

**Phase 2 - NetBox MCP Integration:**
- Comprehensive NetBox MCP server with 142+ tools across DCIM/Virtualization/IPAM/Tenancy
- Performance optimization addressing token overflow and N+1 query issues
- Enterprise-grade safety controls with dry-run capabilities
- Bridget AI assistant with auto-context detection
- Repository: https://github.com/FinnMacCumail/mcp-netbox

## Future Development Plan

A comprehensive four-phase OpenAI orchestration plan exists to replace Claude CLI with modern agentic orchestration:

1. **Phase 1A**: Foundation with GPT-4o-mini intent parsing, LangGraph tool orchestration
2. **Phase 2**: Neo4j graph database integration for instant complex queries  
3. **Phase 3**: RAG-powered semantic intelligence with institutional memory (expanding beyond API endpoint routing)
4. **Phase 4**: Advanced analytics platform using graph algorithms

**Key Performance Targets:**
- 99% cost reduction (from $0.13 to $0.001 per query)
- 15x faster simple queries (3-10s to 200ms-1s)
- 20-50x faster complex queries (30s-3min to 1-3s)

Plan details: `/home/ola/dev/netboxdev/netbox-mcp-docs/openai-agentic-orchestration-plan.md`

## Repository Structure

Complete documentation organization following research best practices:

```
ai-research-2025/
├─ README.md                    # Project overview and links
├─ RESEARCH_LOG.md             # Chronological findings and pivots
├─ CHANGELOG.md                # Conventional changelog for releases
├─ CITATION.cff                # Academic citation metadata
├─ LICENSE                     # Open source license
├─ docs/
│  ├─ index.md                 # Documentation homepage
│  ├─ phases/
│  │  ├─ phase-1-tmdb.md      # TMDB chatbox documentation
│  │  └─ phase-2-netbox-mcp.md # NetBox MCP integration
│  ├─ methods/
│  │  ├─ retrieval-and-planning.md # Core methodologies
│  │  ├─ evaluation.md         # Testing and validation approaches
│  │  └─ tooling.md           # Development tools and setup
│  ├─ adr/
│  │  ├─ 0001-adopt-mkdocs-material.md # Architecture decisions
│  │  └─ 0002-chatbox-architecture.md
│  └─ media/                   # Diagrams, screenshots, demo GIFs
├─ mkdocs.yml                  # MkDocs Material configuration
└─ .github/
   └─ workflows/
      └─ gh-pages.yml          # Automated documentation deployment
```

## Key Research Focus Areas

- Token boundary strategies and pagination handling for large API responses
- Knowledge-graph approaches to reduce N+1 API calls (NetBox VLAN queries: 127 calls → optimized batching)
- Symbolic + semantic retrieval for tool/endpoint selection
- Multi-entity joins and constraint-aware planning with fallback mechanisms
- Institutional memory integration through RAG-powered contextual intelligence

## Reproducibility & Demo Scenarios

**Phase 1 (TMDB) Demo Queries:**
- "Movies starring Leonardo DiCaprio directed by Martin Scorsese"
- "Who created Breaking Bad?"
- Multi-entity constraint solving with progressive relaxation

**Phase 2 (NetBox MCP) Demo Scenarios:**
- "List VLANs" with pagination summary and follow-ups
- "Show interfaces for device X" with summarization and page-through
- "Explain cable relationships for device Y" with structured output
- Complex infrastructure audits demonstrating N+1 query optimization

**Architecture Visualization:**
- Uses Mermaid diagrams for pipeline and architecture flows
- Timeline charts showing research progression and milestones
- Flowcharts demonstrating the core Extraction → Retrieval → Planning → Execution → Validation → Formatting pattern

**Evaluation Approach:**
- Structured testing with realistic data loads
- Performance benchmarks (query response times, cost metrics)
- Validation comparing different approach outcomes
- Reproducible instructions for environment setup and demo execution