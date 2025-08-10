# ADR-0002 — Chatbox Architecture (Planner/Executor)

## Context
A flexible architecture is required to support entity resolution, multi-intent planning, and fallback steps.

## Decision
Use a hybrid approach:
- **Extraction → Retrieval → Planning → Execution → Validation → Formatting**
- Symbolic constraints and provenance-aware fallbacks

## Consequences
- Clear separation of concerns
- Extensible for new domains (TMDB → NetBox)
