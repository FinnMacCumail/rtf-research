# ADR-0029 — LangSmith as the Observability Platform

## Status

**Accepted** — In production use (June 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)

## Context

Through Phase 4, observability in this research programme was conceptual only — the
[Production Considerations](../framework-comparison/production-considerations.md) page described
DIY structlog / Prometheus / OpenTelemetry, but no managed LLM-tracing platform was adopted and no
systematic evaluation existed. Debugging agent behaviour meant fetching traces and reading JSON by
hand.

Phase 5's multi-model, multi-version work made this untenable: comparing 10 models, and gating a
framework upgrade, requires structured, comparable, queryable traces and scores.

## Decision

Adopt **LangSmith** as the observability platform for the DeepAgents build, used for two things:

1. **Tracing** — every agent run is a trace tree (LLM calls, tool calls, middleware steps) with
   tokens, latency, and inputs/outputs per sub-run.
2. **Evaluation substrate** — the model-matrix harness (ADR-0030) records each model run as a
   LangSmith *experiment*, making runs comparable in a leaderboard/comparison view.

## Rationale

- **Trace-level diagnosis is mandatory for agentic failures.** Phase 5 produced a regression
  (a 0.6 default middleware overwriting a correct answer in a post-processing turn) that was
  *invisible at the answer level* and only diagnosable by reading the sub-run chronology. See
  [Phase 5 → Observability](../phases/phase-5-production-deepagents/observability-and-monitoring.md).
- **Native fit.** The agent is built on LangChain/LangGraph/DeepAgents; LangSmith tracing is
  near-zero-integration for that stack.
- **Tracing + evaluation in one platform** closes the detect→diagnose loop without stitching
  tools together.

## Consequences

### Positive
- Regressions are caught by an automated gate and diagnosed from traces.
- Cost/latency attribution per model and per tool call.
- Replaces the hand-written per-run comparison reports.

### Negative / risks
- **Managed-platform dependency** and an API key to manage (a key was leaked into git history
  during this work and had to be scrubbed — see [Lessons Learned](../phases/phase-5-production-deepagents/lessons-learned.md)).
- **Platform failure modes** (expired key, exhausted cloud quota) surface as auth/quota errors in
  the harness; the runner is designed to fail fast rather than emit misleading zeros.
- DIY metrics (Prometheus/OTel) remain appropriate for system-level signals; this ADR covers
  agent-behaviour observability specifically.

## References

- [Research Methods → Observability](../methods/observability.md)
- [Phase 5 → Observability & Monitoring](../phases/phase-5-production-deepagents/observability-and-monitoring.md)
- [ADR-0030 — Model-Matrix Evaluation Harness](0030-model-matrix-evaluation-harness.md)
- [ADR-0024 — Cache Performance Monitoring](0024-cache-performance-monitoring-strategy.md) *(earlier, DIY-monitoring posture)*
