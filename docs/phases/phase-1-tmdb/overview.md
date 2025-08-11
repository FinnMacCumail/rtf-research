# Phase 1 – TMDB Chatbox

## Goals
- Natural-language → structured TMDB queries
- Dynamic endpoint selection (`/discover/movie`, `/person`, etc.)
- Multi-entity joins (e.g., “movies starring A and B”)
- Response formatting: list, timeline, comparison, fact

## Architecture
```mermaid
flowchart TD
  Q[User Query] --> C[Intent/Entity Extractor]
  C --> R[Hybrid Retrieval (semantic+symbolic)]
  R --> P[Planner & Constraint Engine]
  P --> E[Endpoint Executor]
  E --> V[Post-Validation & Rerank]
  V --> F[Formatter (templates)]
  F --> A[Answer]
```

## Repo
- https://github.com/FinnMacCumail/tmdbGPT

## Highlights
- ChromaDB semantic routing: 54 TMDB API endpoint descriptions for intelligent query-to-endpoint matching
- Entity resolution via TMDB Search API calls (people, genres, networks/companies) with parameter mapping
- Constraint-based validation & reranking; flexible response templates
