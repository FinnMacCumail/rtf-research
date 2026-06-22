# Observability & Monitoring

## Why this matters for the research programme

Until Phase 5, observability in this portfolio was discussed only conceptually — the
[Production Considerations](../../framework-comparison/production-considerations.md) page
described DIY structlog / Prometheus / OpenTelemetry, but no managed LLM-tracing platform was
adopted and no systematic evaluation existed. Phase 5 closes that gap: **LangSmith** is used both
for tracing *and* as the substrate for the [model-matrix evaluation harness](multi-model-evaluation.md).

The central claim of this page: **for agentic LLM systems, trace-level observability is not a
"nice to have" — it is the only practical way to find a class of failures that are invisible at
the answer level.** Phase 5 produced a textbook example.

## The observability stack

| Layer | What it does |
|---|---|
| **LangSmith tracing** | Every agent run is a trace; every LLM call, tool call, and middleware step is a nested sub-run with token counts, latency, inputs/outputs. |
| **Evaluation harness** (`tests/eval/`) | Runs a fixed dataset across models, attaches `entity_coverage` / `completeness_judge` / `tool_call_efficiency` scores, and lands each model as a comparable LangSmith *experiment*. |
| **Comparison view** | A sortable leaderboard across experiments — the unit of decision-making, replacing hand-written per-run reports. |

See [Research Methods → Observability](../../methods/observability.md) for the methodology and
[→ Benchmarking](../../methods/benchmarking.md) for the evaluator design.

## Case study — a regression that was invisible at the answer level

The value of trace-level observability is best shown by a real bug it caught during the
DeepAgents 0.6 upgrade.

### Symptom (answer-level)

After upgrading to DeepAgents 0.6.10, the eval harness flagged a regression on the
cross-relationship **VLAN-100** query — the hardest benchmark, whose correct answer is a
*negative finding* ("VLAN 100 is **not** deployed at any of the tenant's sites"). The model's
final answer had collapsed to a useless `"All done. Let me know if you'd like me to look into
anything else."` Completeness scored **0.00**.

A quality-only view says: "the model got worse." That conclusion would have been wrong.

### Diagnosis (trace-level)

Reading the trace's **sub-run chronology** told the real story:

1. The **penultimate** LLM call had produced a *complete, correct* answer — VLAN 100 not deployed,
   with a full breakdown table of the tenant's sites and their actual VLANs.
2. The model then invoked the `write_todos` tool (newly available by default in 0.6).
3. DeepAgents 0.6's `TodoListMiddleware` injects a system-prompt instruction: *"write your final
   answer in the message **after** your last `write_todos` call."*
4. The model dutifully fired one **more** turn to "write the final answer" — but having already
   emitted the substance, it produced only the `"All done…"` filler.
5. **That filler overwrote the good answer as the user-facing output**, and was what the judge scored.

The model never got worse. A new default middleware's prompt instruction **corrupted a correct
answer in a post-processing turn.** This is structurally invisible to anything that looks only at
the final message — you have to see that the *penultimate* call already held the answer.

### Root-cause confirmation

Inspecting the installed framework confirmed the mechanism: 0.6 silently appends ~9.6K characters
of default system-prompt content (`BASE_AGENT_PROMPT` + a sub-agent `task` prompt +
`WRITE_TODOS_SYSTEM_PROMPT`). Two pieces actively harm negative-finding queries — the
"keep iterating until done" framing drives redundant search-hedging, and the
"answer-after-write_todos" rule drives the answer-overwrite.

### Fix

A project `HarnessProfile` that suppresses the harmful defaults:

```python
HarnessProfile(
    base_system_prompt="",                                  # drop BASE_AGENT_PROMPT
    excluded_middleware=frozenset({"TodoListMiddleware"}),  # remove write_todos + its prompt
)
```

Re-running the harness confirmed the VLAN-100 query restored to 1.00 / 1.00, and the aggregate
returned to the pre-upgrade baseline. The fix was *verified by the same observability harness that
caught the regression* — the loop is closed.

## Lessons for observability practice

1. **Score at the answer level; diagnose at the trace level.** The leaderboard tells you *that*
   something regressed; only the sub-run chronology tells you *why*. Both are required.
2. **A regression can come from the framework, not the model.** Pinning the cause to a default
   middleware (not the LLM) depended entirely on seeing the intermediate turns.
3. **The evaluation harness is also the regression detector.** Because every agent change is
   re-scored against a fixed baseline, framework upgrades, prompt edits, and middleware changes
   all get the same automated gate. This is the practical payoff of treating observability as
   infrastructure rather than an afterthought.
4. **Tool-call trajectories are first-class signals**, not just latency noise — see the
   `minimax-m3` retry-loop finding in [Multi-Model Evaluation](multi-model-evaluation.md).

## Contrast with the earlier (Phase 4) observability posture

ADR-0024 and the Production Considerations page treated observability as something you *build*
(structlog, Prometheus, custom hooks). Phase 5's position, validated by the case study above, is
that **managed trace observability + an automated evaluation harness** is a higher-leverage
starting point for agentic systems: it surfaces failure modes (answer-overwrite by a middleware)
that metric counters and log lines would never reveal.

**See also**: [Multi-Model Evaluation](multi-model-evaluation.md) ·
[Lessons Learned](lessons-learned.md) ·
[Research Methods → Observability](../../methods/observability.md) ·
[ADR-0029](../../adr/0029-langsmith-observability-platform.md)
