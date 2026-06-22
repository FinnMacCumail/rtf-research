# Observability

How agent behaviour is made visible and debuggable in this research programme. Introduced in
[Phase 5](../phases/phase-5-production-deepagents/observability-and-monitoring.md); this page is
the reusable methodology.

## Principle

For agentic LLM systems, **the answer is not the unit of observation — the trace is.** A correct
final answer can hide an inefficient or fragile path; an incorrect final answer can hide a correct
intermediate result that was later corrupted. Methodology therefore operates at two levels:

| Level | Question | Instrument |
|---|---|---|
| **Answer level** | Did the run produce a good result? | Evaluators (scores on a fixed dataset) |
| **Trace level** | *Why* did it produce that result? | Per-step sub-run inspection (LLM calls, tool calls, middleware) |

## Tooling: LangSmith

Each agent run is captured as a **trace** — a tree of sub-runs. For each sub-run LangSmith records
inputs/outputs, token counts, latency, and type (LLM / tool / chain). This makes three things
possible that log lines and metric counters do not:

1. **Reconstruct the decision chronology** — e.g. observing that a *penultimate* LLM call already
   held the correct answer before a later turn overwrote it.
2. **Attribute cost** — which model calls and tool calls dominated tokens/latency.
3. **Diff runs** — compare two traces (e.g. before/after a framework upgrade) step by step.

## From tracing to evaluation

Tracing alone is diagnostic; to make it a *gate*, traces are scored. The evaluation harness
(see [Benchmarking](benchmarking.md)) runs a fixed dataset through a target model, attaches
evaluators, and records the result as a LangSmith **experiment**. Because experiments are
comparable, the workflow becomes:

```
change something (model / prompt / middleware / framework version)
        │
        ▼
re-run the harness  ──►  compare experiment vs baseline  ──►  pass / regression
        │                                                        │
        └──────────────── if regression: open the trace ─────────┘
                          and read the sub-run chronology
```

This is the loop that caught (and then verified the fix for) the DeepAgents 0.6
answer-overwrite regression documented in
[Phase 5 → Observability](../phases/phase-5-production-deepagents/observability-and-monitoring.md).

## Practice notes

- **Score to detect, trace to diagnose.** Don't try to infer root cause from scores alone.
- **Treat the harness as the regression detector**, not just a benchmarking tool — every behavioural
  change gets the same automated gate.
- **Keep trace projects per-experiment** so model runs don't cross-contaminate, and so the
  comparison view is clean.
- **Mind the platform's own failure modes** — e.g. an exhausted cloud-inference quota or an expired
  tracing key surfaces as auth/quota errors in the harness, not as bad scores; the runner is
  designed to fail fast and skip-complete rather than silently produce zeros.

## Relationship to the earlier observability posture

Phase 4's [Production Considerations](../framework-comparison/production-considerations.md) framed
observability as something to *build* (structlog, Prometheus, OpenTelemetry, custom hooks). That
remains valid for system-level metrics. The Phase 5 addition is that, for *agent-behaviour*
observability specifically, a managed trace platform plus an automated evaluation harness is the
higher-leverage starting point — it reveals failure modes (a middleware overwriting a correct
answer) that counters and logs cannot.

**See also**: [Evaluation](evaluation.md) · [Benchmarking](benchmarking.md) ·
[ADR-0029 — LangSmith Observability Platform](../adr/0029-langsmith-observability-platform.md)
