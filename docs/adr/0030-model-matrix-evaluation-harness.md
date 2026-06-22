# ADR-0030 — Model-Matrix Evaluation Harness

## Status

**Accepted** — In production use (June 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) (`tests/eval/`)

## Context

Phase 5 needed to compare many models on the same agent task and to gate behavioural changes
(framework upgrades, prompt/middleware edits) against a baseline. The prior workflow — run a query,
read the trace, write a markdown comparison — does not scale to a 10-model matrix and is not a
reliable regression gate.

## Decision

Build a **reproducible model-matrix evaluation harness** on LangSmith:

- **Dataset** (`netbox-benchmark-v2`): four NetBox query classes with machine-checkable
  `expected_entities` ground truth (entity lists, not full reference answers, for stability).
- **Evaluators**: `entity_coverage` (deterministic), `completeness_judge` (LLM-as-judge),
  `tool_call_efficiency` (trajectory). Three complementary signals so neither efficiency
  pathologies nor semantic incompleteness are missed.
- **Runner**: one experiment per model; model-agnostic targets; skip-completed; fail-fast on cloud
  quota; force-rerun flag for deliberate before/after regression checks.

Methodology detail: [Research Methods → Benchmarking](../methods/benchmarking.md).

## Rationale

- **Discriminating ground truth.** An early permissive dataset let a model score 1.0 by naming 3 of
  14 sites; tightening `expected_entities` to full trace-derived ground truth was what made the
  leaderboard meaningful.
- **Three signals beat one.** Quality-only scoring missed a model that reached the right answer via
  18 tool calls (malformed argument shape → validator-bounce retry loop); the trajectory metric
  exposed it.
- **The harness doubles as the regression detector** — every behavioural change is re-scored
  against the baseline, which is how the DeepAgents 0.6 answer-overwrite regression was caught.

## Consequences

### Positive
- Actionable, sortable leaderboard ("capability and cost per query class").
- Automated regression gate for framework/prompt/middleware changes.
- Model-agnostic: local, cloud, and reference models run unchanged.

### Negative / limitations
- **Small fixed dataset** (four queries) — high signal for this domain, but not a broad benchmark;
  results are workload-specific.
- **LLM-as-judge variance** — `completeness_judge` uses a model; judge choice and prompt affect
  scores. Mitigated by pairing it with the deterministic `entity_coverage`.
- **Cloud quota** — a full sweep can exhaust an Ollama Cloud session window; the runner fails fast
  and skip-completes rather than producing zeros.

## References

- [Research Methods → Benchmarking](../methods/benchmarking.md)
- [Multi-Model Evaluation](../phases/phase-5-production-deepagents/multi-model-evaluation.md)
- [ADR-0029 — LangSmith Observability Platform](0029-langsmith-observability-platform.md)
- [Evaluation methodology](../methods/evaluation.md) *(earlier, TMDB-domain evaluation)*
